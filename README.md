# ANSIBLE-CON-NGINX-Y-APACHE2
ANSIBLE CON NGINX Y APACHE2 Y BALANCEO DE CARGA
# Configuración de Servidores Web con Ansible

Este repositorio contiene un playbook de Ansible diseñado para automatizar la configuración de servidores Apache y Nginx. Este documento proporciona una guía completa sobre cómo utilizar los archivos incluidos, así como una explicación de cada componente.

## Tabla de Contenidos

- [Descripción](#descripción)
- [Archivos](#archivos)
- [Requisitos](#requisitos)
- [Estructura del Playbook](#estructura-del-playbook)
  - [main.yml](#main.yml)
  - [apache_setup.yml](#apache_setup.yml)
  - [nginx_setup.yml](#nginx_setup.yml)
- [Ejecución del Playbook](#ejecución-del-playbook)
- [Contribuciones](#contribuciones)
- [Licencia](#licencia)

## Descripción

Este playbook permite configurar múltiples servidores Apache para escuchar en el puerto 8080 y un servidor Nginx que actúa como proxy reverso y balanceador de carga, utilizando SSL para asegurar las conexiones. 

### Funcionalidades

- Instalación y configuración de Apache en múltiples servidores.
- Instalación y configuración de Nginx como proxy reverso.
- Generación de un certificado SSL autofirmado para Nginx.
- Configuración de balanceo de carga entre los servidores Apache.

## Archivos

1. **main.yml**: El playbook principal que coordina las configuraciones de Apache y Nginx.
2. **apache_setup.yml**: Contiene las tareas específicas para la configuración de los servidores Apache.
3. **nginx_setup.yml**: Define las tareas para la configuración del servidor Nginx.

## Requisitos

- **Ansible**: Debes tener Ansible instalado en tu máquina de control.
- **Acceso SSH**: Asegúrate de tener acceso SSH a los servidores donde se realizará la configuración.
- **Sistemas Operativos**: Este playbook está diseñado para sistemas basados en Debian (como Ubuntu).

## Estructura del Playbook

### main.yml

```yaml
---
- name: Playbook principal
  hosts: localhost
  gather_facts: yes
  vars_prompt:
    - name: "apache_ips"
      prompt: "Ingresa las IPs de los servidores Apache, separadas por comas"
      private: no
    - name: "nginx_ip"
      prompt: "Ingresa la IP del servidor Nginx"
      private: no
    - name: "nginx_domain"
      prompt: "Ingresa el nombre de dominio del servidor Nginx (por ejemplo, ejemplo.com)"
      private: no

  tasks:
    - name: Configurar servidores Apache
      include_tasks: apache_setup.yml

    - name: Configurar servidor Nginx
      include_tasks: nginx_setup.yml
```

### apache_setup.yml

```yaml
---
# Configuración de servidores Apache
- name: Convertir las IPs de Apache en una lista
  set_fact:
    apache_servers: "{{ apache_ips.split(',') }}"

- name: Instalar Apache en los servidores especificados
  package:
    name: apache2
    state: present
  delegate_to: "{{ item }}"
  loop: "{{ apache_servers }}"
  become: yes

- name: Configurar Apache para escuchar en el puerto 8080
  lineinfile:
    path: /etc/apache2/ports.conf
    regexp: '^Listen'
    line: 'Listen 8080'
  delegate_to: "{{ item }}"
  loop: "{{ apache_servers }}"
  become: yes

- name: Crear archivo de configuración de Apache para puerto 8080
  copy:
    content: |
      <VirtualHost *:8080>
          DocumentRoot /var/www/html
          ServerAdmin webmaster@localhost
          ErrorLog ${APACHE_LOG_DIR}/error.log
          CustomLog ${APACHE_LOG_DIR}/access.log combined
      </VirtualHost>
    dest: /etc/apache2/sites-available/000-default.conf
  delegate_to: "{{ item }}"
  loop: "{{ apache_servers }}"
  become: yes

- name: Habilitar el sitio de Apache para puerto 8080
  command: a2ensite 000-default.conf
  delegate_to: "{{ item }}"
  loop: "{{ apache_servers }}"
  become: yes

- name: Reiniciar Apache para aplicar cambios
  service:
    name: apache2
    state: restarted
  delegate_to: "{{ item }}"
  loop: "{{ apache_servers }}"
  become: yes
```

### nginx_setup.yml

```yaml
---
# Configuración del servidor Nginx
- name: Instalar Nginx en el servidor Nginx
  package:
    name: nginx
    state: present
  delegate_to: "{{ nginx_ip }}"
  become: yes

- name: Crear directorios para los certificados SSL si no existen
  file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - /etc/ssl/private
    - /etc/ssl/certs
  delegate_to: "{{ nginx_ip }}"
  become: yes

- name: Crear un certificado SSL autofirmado
  command: >
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key
    -out /etc/ssl/certs/nginx-selfsigned.crt
    -subj "/C=US/ST=State/L=City/O=Organization/OU=IT/CN={{ nginx_domain }}"
  delegate_to: "{{ nginx_ip }}"
  become: yes

- name: Configurar Nginx para balanceo de carga con SSL
  copy:
    content: |
      upstream apache_backend {
        {% for ip in apache_servers %}
        server {{ ip }}:8080;
        {% endfor %}
      }

      server {
          listen 80;
          server_name {{ nginx_domain }};
          return 301 https://$host$request_uri;
      }

      server {
          listen 443 ssl;
          server_name {{ nginx_domain }};

          ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
          ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
          ssl_protocols TLSv1.2 TLSv1.3;
          ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';

          ssl_prefer_server_ciphers on;
          ssl_session_cache shared:SSL:10m;
          ssl_session_timeout 1d;

          location / {
              proxy_pass http://apache_backend;
              proxy_set_header Host $host;
              proxy_set_header X-Real-IP $remote_addr;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              proxy_set_header X-Forwarded-Proto $scheme;
          }
      }
    dest: /etc/nginx/sites-available/default
  delegate_to: "{{ nginx_ip }}"
  become: yes

- name: Crear enlace simbólico para habilitar el sitio en Nginx
  file:
    src: /etc/nginx/sites-available/default
    dest: /etc/nginx/sites-enabled/default
    state: link
  delegate_to: "{{ nginx_ip }}"
  become: yes

- name: Reiniciar Nginx para aplicar cambios
  service:
    name: nginx
    state: restarted
  delegate_to: "{{ nginx_ip }}"
  become: yes
```

## Ejecución del Playbook

Para ejecutar el playbook, utiliza el siguiente comando en tu terminal:

```bash
ansible-playbook main.yml
```

### Proceso de Ejecución

1. **Ingreso de Variables**: Al ejecutar el playbook, se te solicitará ingresar las IPs de los servidores Apache, la IP del servidor Nginx y el dominio Nginx.
2. **Configuración de Apache**: Se instalará y configurará Apache en los servidores especificados, incluyendo la configuración para que escuche en el puerto 8080.
3. **Configuración de Nginx**: Se instalará Nginx, se creará un certificado SSL autofirmado y se configurará para actuar como un proxy reverso hacia los servidores Apache.

## Contribuciones

Si deseas contribuir a este proyecto, no dudes en enviar un pull request o abrir un issue. Todas las contribuciones son bienvenidas.

## Licencia

Este proyecto está bajo la Licencia MIT. Puedes usarlo y modificarlo libremente, siempre que se incluya la atribución correspondiente.
