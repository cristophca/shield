# Bienvenidos a la aplicación de SHIELD para Admin-Sistemas

## Instalación del proyecto en local:
1. Haz un fork del proyecto.
2. Descarga el proyecto del repo con git ejecutando el siguiente comando:
```
git clone https://github.com/cristophca/shield.git
```
3. Crea un entorno virtual dentro de la carpeta descargada 'shield':
```
python3 -m venv .venv
```
4. Activa el entorno virtual:
```
source .venv/bin/activate
```
5. Instala las librerías necesarias que se encuentran en el fichero `requirements.txt`:
```
pip install -r requirements.txt
```
6. Ejecuta las migraciones:
```
python manage.py migrate
```
Si queremos observar todas las migraciones que se han efectuado ejecuta lo siguiente:
```
python manage.py makemigrations
```
7. Carga los datos de superheroes del fichero `superheroes.csv` usando el comando `metahumans/management/commands/load_from_csv.py`. Si os da problemas usando el comando load_from_csv, podéis usar el comando `loaddata` con el fichero `metahumans/fixtures/initial_data.json` 
```
python manage.py loaddata metahumans/fixtures/initial_data.json
```
8. Crea tu propio usuario superuser para poder entrar en el admin de django:
```
python manage.py createsuperuser
```
Escriba su nombre de usuario (en minúscula, sin espacios), dirección de correo electrónico y contraseña cuando te pregunten por ellos.
```
Username: admin
Email address: admin@admin.com
Password:
Password (again):
Superuser created successfully. 
```

9. Ejecuta el servidor de django para probar la aplicación.
```
python manage.py runserver
```

A continuación, se detalla la información necesaria para poder desplegar la aplicación con Fabric, Ansible y Docker en cualquier máquina remota con sistema operativo Ubuntu (Vagran o servidor AWS).

## Despliegue del  proyecto en una máquina remota con Fabric.
### Instalaciones necesarias:
Instalar la libreria de fabric:
```
pip install fabric
```
### Crea del fichero de trabajo `fabfile.py` y comprueba la conexión con el servidor remoto:

Crea un fichero llamado `fabfile.py` con el siguiente contenido:
```python
from fabric import Connection, task

@task
def development(ctx):
    ctx.user = 'vagrant'
    ctx.host = '192.168.33.10'
    ctx.connect_kwargs = {"password": "vagrant"}

@task
def deploy(ctx):
    with Connection(ctx.host, ctx.user, connect_kwargs=ctx.connect_kwargs) as conn:
        conn.run("uname")
        conn.run("ls")
```
(Con las funciones anteriores aparecerá la distribución sobre la que corre la aplicación y los ficheros no ocultos que se tiene en el local de la máquina remota).

### Despliegue: Contenido completo  del fichero `fabfile.py` para que instale el proyecto shield en una máquina remota con Fabric.
1. Se añaden los import necearios:
```python
import sys
import os
from fabric import Connection, task
```
2. Se definen las variables que nos servirán como constantes en el despliegue de la aplicación:
```python
PROJECT_NAME = "shield"
PROJECT_PATH = f"~/{PROJECT_NAME}"
REPO_URL = "https://github.com/cristophca/shield.git"
VENV_PYTHON = f'{PROJECT_PATH}/.venv/bin/python'
VENV_PIP = f'{PROJECT_PATH}/.venv/bin/pip'
```
3. Para instalar el proyecto en el servidor:
- Se crea la conexión:
```python
@task
def development(ctx):
    ctx.user = 'vagrant'
    ctx.host = '192.168.33.10'
    ctx.connect_kwargs = {"password": "vagrant"}

def get_connection(ctx):
    try:
        with Connection(ctx.host, ctx.user, connect_kwargs=ctx.connect_kwargs) as conn:
            return conn
    except Exception as e:
        return None

```
- Tarea de `git clone`: Clonar repositorio
```python
@task
def clone(ctx): 
    print(f"clone repo {REPO_URL}...")   
    
    if isinstance(ctx, Connection):
        conn = ctx
    else:
        conn = get_connection(ctx)

    # obtengo las carpetas del directorio (Llamar al método rum)
    ls_result = conn.run("ls").stdout

    # divido el resultado para tener cada carpeta en un objeto de una lista
    ls_result = ls_result.split("\n")

    # si el nombre del proyecto ya está en la lista de carpetas
    # no es necesario hacer el clone 
    if PROJECT_NAME in ls_result:
        print("project already exists")
    else:
        conn.run(f"git clone {REPO_URL} {PROJECT_NAME}")


```
- Tarea `checkout` de la rama `main` del repositorio: 
```python
@task
def checkout(ctx, branch=None):
    print(f"checkout to branch {branch}...")

    if branch is None:
        sys.exit("branch name is not specified")
    
    if isinstance(ctx, Connection):
        conn = ctx
    else:
        conn = get_connection(ctx)
    
    with conn.cd(PROJECT_PATH):
        conn.run(f"git checkout {branch}")

```
- Tarea `git pull`:
```python
@task
def pull(ctx, branch="main"):

    print(f"pulling latest code from {branch} branch...")

    if branch is None:
        sys.exit("branch name is not specified")

    if isinstance(ctx, Connection):
        conn = ctx
    else:
        conn = get_connection(ctx)

    with conn.cd(PROJECT_PATH):
        conn.run(f"git pull origin {branch}")

```
- Tarea `venv` (Crear entorno virtual):
```python

@task
def create_venv(ctx):
    
    print("creating venv....")
    
    if isinstance(ctx, Connection):
        conn = ctx
    else:
        conn = get_connection(ctx)
    with conn.cd(PROJECT_PATH):
        conn.run("python3 -m venv .venv")
        conn.run(f"{VENV_PIP} install -r requirements.txt")

```
-Tarea `migrate`:Ejecutar las migraciones
```python
@task
def migrate(ctx):
    print("checking for django db migrations...")

    if isinstance(ctx, Connection):
        conn = ctx
    else:
        conn = get_connection(ctx)
        
    with conn.cd(PROJECT_PATH):
        conn.run(f"{VENV_PYTHON} manage.py migrate")

```

-Tarea `loaddata`:Cargar los datos de inicio
```python
@task
def loaddata(ctx):
    print("loading data from fixtures...")

    if isinstance(ctx, Connection):
        conn = ctx
    else:
        conn = get_connection(ctx)

    with conn.cd(PROJECT_PATH):
        conn.run(f"{VENV_PYTHON} manage.py loaddata metahumans/fixtures/initial_data.json")
```
-Se añaden todas las tareas anteriores al proceso deploy:
```python
@task
def deploy(ctx):
    conn = get_connection(ctx)
    if conn is None:
        sys.exit("Failed to get connection")

    clone(conn)
    checkout(conn, branch="main")
    pull(conn, branch="main")
    create_venv(conn)
    migrate(conn)
    loaddata(conn)
```
 4. Ejecuta el script con el siguiente comando :
 
```
fab development deploy  
```


## Despliegue del  proyecto en una máquina remota con Ansible.
### Instalaciones necearias:
0. Instala Ansible en tu máquina local (Terminal de Ubuntu).
Para Ubuntu:
```
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```
Si se desea comprobar que se instaló correctamente ejecuta el siguiente comando:
```
ansible localhost -m ping --ask-pass
```
Puedes visitar https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html
### Crea la carpeta de trabajo  `ansible`: 
1. Dentro de la aplicación shield, crea la carpeta "ansible".
### Despliegue: Contenido completo  de la carpeta `ansible` para que instale el proyecto shield en una máquina remota con Ansible.
2. La carpeta "ansible" debe contener los siguiente ficheros:
- Fichero nombrado "hosts": En este fichero ubicaremos el inventario. 

- Fichero nombrado "vars.yml":Este fichero contendrá las variables de la máquina.

- Fichero nombrado "provision.yml": Es el ejecutable para provisionar la máquina (librerias, etc).
El fichero `provision.yml`se debe ejecutar con el siguiente comando:

```
 ansible-playbook -i hosts provision.yml --user=cristopher --ask-pass --ask-become-pass
```
Si usas AWS :
```
ansible-playbook -i hosts provision.yml --user=cristopher --ask-pass --ask-become-pass
```

- Fichero nombrado "deploy.yml": Contendrá el despliegue de la aplicación en python.
```
 ansible-playbook -i hosts deploy.yml --user=cristopher --ask-pass --ask-become-pass
```
Si usas AWS :
```
ansible-playbook -i hosts deploy.yml --user=vagran --ask-pass --ask-become-pass
```

Puede visitar https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html  para completar los distintos ficheros anteriormente nombrados.

## Despliegue del  proyecto en una máquina remota con Docker.
### Instalaciones necearias:
0. Instala Ansible en tu máquina local.
- Para Windows (WSL 2): https://docs.docker.com/docker-for-windows/wsl/
- Para Mac:
 https://docs.docker.com/docker-for-mac/install/
- Para Ubuntu: 
https://docs.docker.com/engine/install/ubuntu/

### Crea del fichero de trabajo `Dockerfile` en la raíz del proyecto que debe contener:
1. Heredar el comportamiento base de la imagen `python:3.9-slim-buster`:
```
FROM python:3.9-slim-buster
```
Puede visitar https://hub.docker.com/_/python para más informacion.

2. Especificar el directorio de trabajo:
```
WORKDIR /app
```
3. Prepararel entorno de ejecución (Copia de fichero `requirements.txt` a la imagen):
```
COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt
```
4. Copiar el conternido del directorio a la imagen de Docker:
```
COPY . .
```
5. Arrancar la aplicación Shield:
```
CMD [ "python3", "manage.py" , "runserver" , "0.0.0.0:8000"]
```
