# Sistemas Distribuidos, Parcial 1

## Juliana Zúñiga-A00068120

Comandos de linux necesarios para el aprovisionamiento de un ambiente compuesto por los siguientes elementos: Un servidor encargado de almacenar logs por medio de la aplicación Elasticsearch, un servidor encargado de hacer la conversión de logs por medio de la aplicación Logstash, un servidor con la herramienta encargada de visualizar la información de los logs por medio de la aplicación Kibana y uno o varios servidores web ejecutando la aplicación FileBeat para el envío de los logs al servidor con Logstash


## Elastic Search

Este es el servicio encargado de almacenar los logs enviados por los clientes. Los pasos necesarios para su instalación son los siguientes:

1. Agregar el repositorio en el cual se encuentra ElasticSearch para CentOS.

Para ello se agregan las siguientes líneas de código en la ruta `/etc/yum.repos.d/elasticsearch.repo`

```bash
[elasticsearch]
name=Elasticsearch repository
baseurl=http://packages.elastic.co/elasticsearch/2.x/centos
gpgcheck=1
gpgkey=http://packages.elastic.co/GPG-KEY-elasticsearch
enabled=1
```

2. Ejecutar los siguientes comandos:

```bash
yum install java
rpm --import http://packages.elastic.co/GPG-KEY-elasticsearch
yum install elasticsearch
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch
```
3. Abrir el puerto de escucha de ElasticSearch mediante firewalld, para ello se deben ejecutar los siguientes comandos:

```
systemctl start firewalld
firewall-cmd --add-port=9200/tcp
firewall-cmd --add-port=9200/tcp --permanent
```
4. Para probar el buen funcionamiento de ElasticSearch, ingresar a [ip-elasticsearch]:9200 y debería obtener como resultado lo siguiente:

```
{
  "name" : "Captain Zero",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "A9MBArhyS82Qh8tF25UR7Q",
  "version" : {
    "number" : "2.4.6",
    "build_hash" : "5376dca9f70f3abef96a77f4bb22720ace8240fd",
    "build_timestamp" : "2017-07-18T12:17:44Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.4"
  },
  "tagline" : "You Know, for Search"
}
```

## LOGSTASH

Este es el servicio encargado de procesar los logs almacenados en el servidor de ElasticSearch. Los pasos necesarios para su instalación son los siguientes:

1. Agregar el repositorio en el cual se encuentra LogStash para CentOS.

Se agregan las siguientes líneas de código en la ruta `/etc/yum.repos.d/logstash.repo`

```bash
[logstash]
name=Logstash
baseurl=http://packages.elasticsearch.org/logstash/2.2/centos
gpgcheck=1
gpgkey=http://packages.elasticsearch.org/GPG-KEY-elasticsearch
enabled=1
```
2. Ejectutar el siguiente comando:

```bash
yum -y install logstash
```
3. Una vez instalado, se debe agregar la dirección ip en la cual esta corriendo LogStash en la siguiente ruta: `/etc/pki/tls/openssl.cnf`

```
[ v3_ca ]
subjectAltName = IP: 192.168.133.13
```
4. Generar un certificado SSL con el objetivo de que los logs se transfieran de manera segura.

```bash

cd /etc/pki/tls

#openssl req -config /etc/pki/tls/openssl.cnf -x509 -days 3650 -batch -nodes -newkey rsa:2048 -keyout private/logstash-forwarder.key -out certs/logstash-forwarder.crt

```

5. Crear el archivo input.conf en la ruta  `/etc/logstash/conf.d/` con el siguiente contenido:

```
input {
beats {
port => 5044
ssl => true
ssl_certificate => "/etc/pki/tls/certs/logstash-forwarder.crt"
ssl_key => "/etc/pki/tls/private/logstash-forwarder.key"
}
}

```

6. Crear el archivo output.conf en la ruta `/etc/logstash/conf.d/` con el siguiente contenido:

```
output {
elasticsearch {
hosts => ["192.168.133.12:9200"]
sniffing => true
manage_template => false
index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
document_type => "%{[@metadata][type]}"
}
}

```

7. Una vez agregados los dos archivos anteriores, se debe crear el archivo filter.conf en la ruta `/etc/logstash/conf.d/` con el siguiente contenido:

```
filter {
if [type] == "syslog" {
grok {
match => { "message" => "%{SYSLOGLINE}" }
}
date {
match => [ "timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
}
}
}

```

8. Verificar que la configuración esté correcta e iniciar el servicio usando los comandos:

```bash
service logstash configtest
systemctl daemon-reload
systemctl start logstash
systemctl enable logstash
systemctl start firewalld
firewall-cmd --add-port=5044/tcp
firewall-cmd --add-port=5044/tcp --permanent
```


## KIBANA

1. Agregar el repositorio en el cual se encuentra Kibana para CentOS.

Agregar las siguientes líneas de código en la ruta `/etc/yum.repos.d/kibana.repo`

```
[kibana]
name=Kibana repository
baseurl=http://packages.elastic.co/kibana/4.4/centos
gpgcheck=1
gpgkey=http://packages.elastic.co/GPG-KEY-elasticsearch
enabled=1
```

2. Ejecutar los siguientes comandos:

```bash
yum install kibana
systemctl daemon-reload
systemctl start kibana
systemctl enable kibana
systemctl start firewalld
firewall-cmd --add-port=5601/tcp
firewall-cmd --add-port=5601/tcp --permanent
```
## FileBeat

Los comandos que se deben ejecutar para instalar el cliente FileBeat son los siguientes:

```bash
yum install filebeat
systemctl start filebeat
systemctl enable filebeat
```

## VAGRANTFILE

```bash
# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.ssh.insert_key = false
  config.vbguest.auto_update = false
  config.vm.define :elastic do |es|
    es.vm.box = "centos1706_v0.2.0"
    es.vm.hostname = "kratos"
    es.vm.network "private_network", ip: "192.168.133.12"
    es.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "1024","--cpus", "1", "--name", "centos_elastic" ]
    end

  es.vm.provision :chef_solo do |chef|
    chef.install = false
    chef.cookbooks_path = "cookbooks"
    chef.add_recipe "elastic"
  end
 
end

  config.vm.define :log do |log|
    log.vm.box = "centos1706_v0.2.0"
    log.vm.hostname = "venus"
    log.vm.network "private_network", ip: "192.168.133.13"
    log.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "1024","--cpus", "1", "--name", "centos_logstash" ]
    end

  log.vm.provision :chef_solo do |chef|
    chef.install = false
    chef.cookbooks_path = "cookbooks"
    chef.add_recipe "logstash"
  end
 
end

  config.vm.define :kibana do |kiba|
    kiba.vm.box = "centos1706_v0.2.0"
    kiba.vm.hostname = "zeus"
    kiba.vm.network "private_network", ip: "192.168.133.14"
    kiba.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "1024","--cpus", "1", "--name", "centos_kibana" ]
    end

  kiba.vm.provision :chef_solo do |chef|
    chef.install = false
    chef.cookbooks_path = "cookbooks"
    chef.add_recipe "kibana"
  end
 
end

  config.vm.define :filebeat do |file|
    file.vm.box = "centos1706_v0.2.0"
    file.vm.hostname = "terminator"
    file.vm.network "private_network", ip: "192.168.133.15"
    file.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "1024","--cpus", "1", "--name", "centos_file_beat" ]
    end

  file.vm.provision :chef_solo do |chef|
    chef.install = false
    chef.cookbooks_path = "cookbooks"
    chef.add_recipe "filebeat"
  end
 
end




```
## COOKBOOKS

| Directorio     | Descripción     |
| ------------- | ------------- |
| cookbooks/elastic/| Contiene los archivos y las instrucciones que se desean automatizar para la implementacion de elasticsearch. Las carpetas principales que contiene este directorio son los recipes y los files. En el primero se encuentra todas las lineas que se quieren automatizar y las que permiten agregar los archivos de la carpeta files. |
| cookbooks/logstash/ | Contiene los archivos e instrucciones que se desean automatizar para el buen funcionamiento del servidor logstash. En la carpeta files de esta ruta, podemos encontrar los archivos necesarios para la implementacion del servidor logstash |
| cookbooks/kibana/ | Contiene los archivos e instrucciones que se desean automatizar para el buen funcionamiento del servidor kibana. En la carpeta files de esta ruta, podemos encontrar los archivos necesarios para la implementacion del servidor.|
| cookbooks/filebeat/ | Contiene los archivos e instrucciones que se desean automatizar para una correcta configuración del cliente que nos brindara los logs. En la carpteta files de esta ruta se encuentra los archivos necesarios para la configuración del cliente.|

## Evidencia del buen funcionamiento del sistema:

![][1]

[1]: img/img1.png


En la imagen anterior se muestran los logs del servicio FileBeat corriendo en el host terminator.

URL: https://github.com/julizuoso/sd-exam1.git