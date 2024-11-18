# Scripts de conexión a través de AWS CLI SSM

Repositorio para simplificar la conexión con recursos EC2 y RDS en la infraestructura de AWS mediante conexiones con AWS CLI.

## Requisitos

1. Tener [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) instalado.
2. Tener activo el [Session Manager Plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html).
3. Tener llaves para configurar los perfiles en el aws cli `aws configure --profile <profile_name>`

## Configuración de aliases

Para establecer las conexiones se deben definir aliases mediante el archivo `connect_aliases.ini`. En el repositorio se encuentra el archivo `connect_aliases.ini.sample` al que se puede sacar una copia y configurar los IDs de las instancias, así como los host y puertos para la configuración de conexiones de base de datos.

Para cada una de las conexiones con base de datos, se debe configurar los siguientes alias relacionados con el ambiente:

- `db-<env_name>-instance` : ID de la instancia que se va a usar como puente para la conexión. Esta instancia debe tener SG configurado para interactuar con la DB. Ejemplo, `i-0fa7da8f84403afff`.
- `db-<env_name>-host` : El host al que se van a enviar las conexiones de base de datos. Ejemplo, `sample-database.us-east-1.rds.amazonaws.com`.
- `db-<env_name>-port` : Puerto para establecer conexión con el host de la base de datos. Ejemplo, `3306`.
- `db-<env_name>-local-port` : Puerto de la máquina local que se va a utilizar para la conexión con la base de datos (Puerto para hacer forward). Ejemplo `3307`.

En caso que se manejen diferentes perfiles de AWS CLI, se recomienda definir un prefijo para el establecimiento de las conexiones. Este prefijo se define mediante la variable `aws-profile-prefix` y es utilizado para las conexiones. Por ejemplo, si el prefijo es `sample`, al establecer una conexión para el ambiente `dev` se establecerá con el perfil llamado `sample-dev`.

Para el establecimiento de sesiones remotas mediante SSH a través del CLI, se deben definir las rutas a los key file configurados como **keypair** en las instancias. Para ello se debe guardar la ruta correspondiente a cada ambiente en las variables con el siguiente formato: `ssh-<env_name>-key` (Ejemplo, `ssh-dev-key=keys/keyfile.pem`). Se recomienda crear una carpeta llamada `keys` en la raíz de este repo para almacenar los key files. Además se debe agregar la configuración de ProxyCommand en el archivo `config` de SSH (usualmente ubicado en `~/.ssh/config`):

~~~ssh
# SSH over Session Manager
host i-* mi-*
    ProxyCommand aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters portNumber=%p
~~~

Adicionalmente, para las conexiones remotas con las máquinas, se puede crear alias para los ID de las instancias, facilitando la recordación de conexiones, para ello se debe seguir el siguiente patrón de nombramiento: `<env_name>-<instance_id_alias>=<instance_id>`. (Ejemplo: `dev-web-1=i-0fa7da8f84403afff`, esto apunta a una conexión con un alias `web-1` para el ambiente `dev` que apuntaría a la máquina con ID `i-0fa7da8f84403afff`).

## Scripts

`bin/connect [-t <connect_type>] [-u <connect_user>] <env_name> <target>`

- Conexión con instancias EC2.
- El parámetro opcional `-t` permite indicar si se quiere establecer una conexion SSH a través de SSM.
- El parámetro opcional `-u` permite establecer el usuario que se va a usar para la conexión SSH. Por defecto se usa el usuario `ubuntu`.
- El `<env_name>` puede ser solamente `dev`, `qa` o `pdn`.
- El `<target>` puede ser un ID de instancia o un alias definido para uno en el archivo `connect_aliases.ini`.

`bin/db-connect <env_name>`

- Abrir puerto para conexión con RDS.
- El `<env_name>` puede ser solamente `dev`, `qa` o `pdn`.

## Conectar mediante la extensión de VSCode llamada [Remote-SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)

Para establecer sesiones interactivas a través de la extensión de VSCode **Remote-SSH**, es necesario haber establecido la configuración para conexiones SSH a través de SSM como se indica en el apartado de **Configuración de aliases**. Además es necesario configurar los hosts mediante el ProxyCommand:

~~~ssh
Host <connection_name>
    ProxyCommand aws ssm start-session --profile <aws_profile> --target <instance_id> --document-name AWS-StartSSHSession --parameters portNumber=%p
    User <ssh_user>
    IdentityFile <path_to_key_file>
~~~

- `<connection_name>` : Nombre que se va a usar en VSCode para identificar la conexión. Ejemplo, `sample-ec2`.
- `<aws_profile>` : Perfil de AWS CLI que se va a usar para establecer la conexión. Ejemplo, `sample-dev`.
- `<instance_id>` : ID de la instancia con la que se va a establecer la conexión. Ejemplo, `i-0fa7da8f84403afff`.
- `<ssh_user>` : Usuario que se va a usar en la autenticación de la conexión SSH. Ejemplo, `ubuntu`.
- `<path_to_key_file>` : Ruta absoluta al key file usado para la autenticación de la conexión SSH. Ejemplo, `C:\Users\win_user\keys\keyfile.pem`.

## Consideraciones para usar con Windows-WSL

Si para el desarrollo se utiliza WSL, se recomienda realizar las siguientes configuraciones y consideraciones:

1. Instale y configure el AWS CLI a nivel de Windows.
2. Use los scripts de este repositorio a través de la terminal de Git Bash que tendrá disponible una vez haya instalado [Git](https://git-scm.com/downloads/win).
3. Si va a usar conexiones SSH a través de SSM configure a nivel de Windows el ProxyCommand como se indica en el apartado de **Configuración de aliases**.
4. Instale el AWS CLI y el Session Manager Plugin a nivel de WSL para usarlo en ese entorno.
5. En caso que quiera compartir la configuración de perfiles de AWS CLI entre los entornos Windows y WSL; cree un enlace simbólico de la carperta `.aws` en el sistema de archivo Windows al sistema de archivos Linux use `ln -v -s /mnt/c/Users/<windows_user>/.aws /home/<wsl_user>/`.
6. En caso de querer compartir los key files para establecer conexiones SSH a través de SSM dentro del entorno WSL, estas se deben hacer mediante el usuario `root`, por tanto debe compartirse igual la configuración de AWS CLI para minimizar problemas de mantenibilidad usando `ln -v -s /mnt/c/Users/<windows_user>/.aws /root/` y configurar el ProxyCommand de SSH en el archivo `/root/.ssh/config`. Una vez configurado podrá conectarse de la siguiente manera `AWS_DEFAULT_PROFILE=<aws_profile> ssh -i <path_to_key_file> <ssh_user>@<instance_id>`.
