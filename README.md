# Instalar Docker Nagios, esta version Vale para X32, x64 con graficas

        cd /opt

        sudo git clone https://github.com/txurtxil/NagiosGraf/

       cd nagios
   
       sudo docker-compose -p nagios up -d    (nagios es el nombre que le damos al contenedor)

       Con estos pasos tenemos nagios en la url http://IP_del_host:9080 
   
      Ver en "localhost:8080"

      Usuario: nagiosadmin

      Password: nagios
        
# Configurar Nagios:

### Entrar en la Shell del volumen Nagios para poder trabajar con los ficheros de configuracion Nagios

    sudo docker exec -it nagios_nagios_1 /bin/bash

      Ubicacion nagios:
      
      /opt/nagios
      
      /opt/nagios/libexdec ---> scripts
      
      /opt/nagios/etc ----> Ficheros configuracion:

         •localhost.cfg: Plantilla de configuración para monitorizar los servicios del propio servidor Nagios.

         • hypervisor.cfg: Plantilla de configuración para monitorizar los servicios de las plataformas de virtualización.

         • linux.cfg: Plantilla de configuración para monitorizar los servicios de los hosts Linux.

         • windows.cfg: Plantilla de configuración para monitorizar los servicios de los hosts Windows.

         • netdev.cfg: Plantilla de configuración para monitorizar los servicios de los dispositivos de red.

# Monitorizar Servidor Windows en Nagios

1.Instalar el agente en los hosts Windows a monitorizar, en nuestro caso instalaremos “NSClient++”  x64,   a descargar free desde su web:

             1.Generic
             2.Complete
             3. IP equipo a monitorizar (copiamos la password) y activar: 
                Enable check plugins, Enable nsclient server, Enable NRPE server,insecure legacy
             4. Hay que editar el fichero C:\Program Files\NSClient++\nsclient.ini y poner "enabled" los checks 

2. Creamos una carpeta donde alojaremos el fichero .cfg de maquinas a monitorzar

   mkdir /opt/nagios/etc/servers

3. copiamos la plantilla /opt/nagios/etc/objects/windows.cfg del equipo a minitorizar:
   
    cp /opt/nagios/etc/objects/windows.cfg /opt/nagios/etc/objects/equipos/win01.cfg
   
  Editamos el nuevo fichero win01.cfg y en  el apartado "define host" poner la direccion IP del equipo a monitorizar y sustituimos en todo el documeto la definicion que viene de serie en host_name (winserver) por el nombre que que damos a lequipo
  a monitorizar.

4.Editamos el fichero de configuración de nagios:

    nano  /opt/nagios/etc/nagios.cfg 
    
Le decimos use la siguiente carpeta y añada todos los ficheros a monitorizar (seran servidores y equipos) "You can also tell Nagios to process all config files with a .cfg" quitamos almoadilla
y salvamos:

        cfg_dir=/opt/nagios/etc/servers
        
6.Editamos el fichero donde se definen los comandos para nagios:

         nano /opt/nagios/etc/objects/commands.cfg

En la definición del comando para “check_nt” es donde se situa la comunicacion con el equipo y lleva la password generada durante la instalacion de “NSClient++” 
, comentamos el que viene por defecto y agregamos el siguiente:

        command_line $USER1$/check_nt -H $HOSTADDRESS$ -p 12489 -s YPg7gMkLOOne49PB -v $ARG1$ $ARG2$

Con este procedimiento, despues de reiniciar Nagios,  ya deberiamos tener monitorizado el primer equipo windows

## Comandos utiles para tranajar con el docker nagios
### Reiniciar Nagios para aplicar cambios (desde el equipo host): 

     sudo docker restart nagios_nagios_1

## Eliminar Docker Nagios

cd nagios

sudo docker-compose down

sudo docker rmi  71c4992638a2 (poner Image ID: sudo docker image ls)

## Para probar la configuracion de Nagios antes de aplicar las modificaciones en lso servidores:

    nagios -v nagios.cfg

## Para Monitorizar SRV REL 6.X usaremos el agente nagios NCPA https://www.nagios.org/ncpa/archive.php

            -Bajar el agente ncpa-2.1.8.el6.x86_64.rpm,
            -instalarlo con rpm -vih ncpa-2.1.8.el6.x86_64.rpm
            -Efitamos el fichero de configuracion del agente para cambiar el toque generico "mytoken"
               nano /usr/local/ncpa/etc/ncpa.cfg
            -Podemos lanzar/reinicar este servicio con:
                 service ncpa_listener restart
            -Tenemos el servicio funcionado en la URL  
                       https://localhost:5693/
              si no hemos cambiado el token sera mytoken
## Desde naguios el check a usar de libexec es check_ncpa.py
               -Hay algun error en este docker en la ejecucion de este script se soluciona asi:
                        sudo ln -s /usr/bin/python3 /usr/bin/python
               -Comando para saber el espacio usado de la unidad /mnt/datos del servidro RHEL 6.x:            
               /opt/nagios/libexec/check_ncpa.py -H 172.16.44.72 -t mytoken -M 'disk/logical/|mnt|datos/used_percent' --warning 90 --critical 95
                         OK: Used_percent was 15.50 % | 'used_percent'=15.50%;90;95;

## Hablilitar los checks NCPA en Nagios: 

                  -Editrar el fichero commands.cfg y añadir la siguiente linea:
                  
                         nano /opt/nagios/etc/objects/commands.cfg
                         
                           define command {
                                        command_name    check_ncpa
                                        command_line    $USER1$/check_ncpa.py -H $HOSTADDRESS$ $ARG1$
                                          }
                                          
                    -Creamos el siguiente fichero de ejemplo para monitorizar un servidor RHEL

                        nano /opt/nagios/etc/servers/equipoRHEL6x.cfg
                    
                    -Añadimos los check para monitorizar Momeria, CPU, discos de un servidor RHEL 6.x
                                       
                              
                           
                           define host {
                               host_name               equipoRHEL6x
                               address                 172.16.44.72
                               check_command           check_ncpa!-t 'mytoken' -P 5693 -M system/agent_version
                               max_check_attempts      5
                           }
                           
                           define service {
                               use                     generic-service,graphed-service
                               host_name               equipoRHEL6x
                               service_description     CPU Usage
                               check_command           check_ncpa!-t 'mytoken' -P 5693 -M cpu/percent -w 20 -c 40 -q 'aggregate=avg'
                               max_check_attempts      5
                           }
                           
                           
                           define service {
                               use                     generic-service,graphed-service
                               host_name               equipoRHEL6x
                               service_description     Memory Usage
                               check_command           check_ncpa!-t 'mytoken' -P 5693 -M memory/virtual -w 50 -c 80 -u G
                               max_check_attempts      5
                           }
                           
                           
                           define service {
                               use                     generic-service,graphed-service
                               host_name               equipoRHEL6x
                               service_description     Process Count
                               check_command           check_ncpa!-t 'mytoken' -P 5693 -M processes -w 150 -c 200
                           }
                           
                           
                           define service {
                               use                     generic-service,graphed-service
                               host_name               equipoRHEL6x
                               service_description     Unidad de sistema Operativo
                               check_command           check_ncpa!-t 'mytoken' -P 5693 -M 'disk/logical/|/used_percent' --warning 90 --critical 95
                           }
                           
                           
                           define service {
                               use                     generic-service,graphed-service
                               host_name               equipoRHEL6x
                               service_description     Unidad de Datos Alfresco
                               check_command           check_ncpa!-t 'mytoken' -P 5693 -M 'disk/logical/|mnt|datos/used_percent' --warning 90 --critical 95
                           }



## Comprobaciones que ya ha cogido la configuracion correcamente:

                  nagios -v nagios.cfg

                  Si todo ha ido ok reinciar nagios desde el host:

                  sudo docker restart nagios_nagios_1


## Para Monitorizar SRV Windows XX  usaremos el agente nagios NCPA, mejo que el agente NSCP++ mismo procedimeinto que en linux y mismo fichero check de Nagios

            ncpa-X.X.X .exe

       




            






















                              
                    

                              
                    



 







