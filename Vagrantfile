# ========================================================================================================================================
# ================== CONFIGURAÇÃO DO BANCO DE DADOS ======================================================================================
# ========================================================================================================================================

Vagrant.configure("2") do |config|
  config.vm.define :db do |db_config|
    db_config.vm.box = "bento/ubuntu-22.04"                             
    db_config.vm.hostname = "postgresql-server"
    db_config.vm.network :private_network, ip: "192.168.56.11"              # IP do servidor

    db_config.vm.synced_folder ".", "/vagrant", type: "virtualbox"

    # Especificações da máquina virtual
    db_config.vm.provider "virtualbox" do |vb|
      vb.name   = "postgresql-server"
      vb.memory = "1024" # 1 GB RAM
      vb.cpus   = 1      # 1 CPU 
    end

    # Scrpit executado quando a máquina é criada
    db_config.vm.provision "shell", inline: <<-SHELL

      # Atualiza os pacotes e instala o PostgreSQL
      apt-get update -y
      apt-get upgrade -y
      apt-get install -y postgresql

      # Configrura o servidor para receber requisições de outros IPs
      sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" /etc/postgresql/*/main/postgresql.conf

      cat >> /etc/postgresql/*/main/pg_hba.conf <<EOF
host    all             all             0.0.0.0/0               scram-sha-256
host    all             all             ::/0                    scram-sha-256
EOF

      DB_NAME="db_auth"             # Nome do database
      DB_ROOT_PASS="12345"          # Senha do usuário

      # Altera a senha do usuário "postgres" para a senha definida 
      sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD '${DB_ROOT_PASS}';"

      # Cria um database 
      sudo -u postgres psql -c "CREATE DATABASE ${DB_NAME};"

      # Reinicia o service
      systemctl restart postgresql

      # Executa um scrpit SQL 
      sudo -u postgres psql -f /vagrant/sources/init.sql
    SHELL
  end

# ========================================================================================================================================
# ================== CONFIGURAÇÃO DO TOMCAT ==============================================================================================
# ========================================================================================================================================

  config.vm.define :web do |web_config|
    web_config.vm.box = "bento/ubuntu-22.04"
    web_config.vm.hostname = "tomcat-server"
    web_config.vm.network :private_network, ip: "192.168.56.10"           # IP do servidor
    web_config.vm.network "forwarded_port", guest: 8080, host: 8080       # Expõe a porta 8080 para HTTP
    web_config.vm.network "forwarded_port", guest: 8443, host: 8443       # Expõe a porta 8443 para HTTPS

    web_config.vm.synced_folder ".", "/vagrant", type: "virtualbox"

    # Especificações da máquina virtual
    web_config.vm.provider "virtualbox" do |vb|
      vb.name   = "tomcat-server"
      vb.memory = "1664" # 1.6 GB RAM
      vb.cpus   = 1      # 1 CPU
    end

    # Scrpit executado quando a máquina é criada
    web_config.vm.provision "shell", inline: <<-SHELL
      # Atualiza os pacotes
      apt-get update -y
      apt-get upgrade -y
      apt-get install -y wget tar maven

      JDK_URL="https://api.adoptium.net/v3/binary/latest/25/ga/linux/x64/jdk/hotspot/normal/eclipse"        # URL da JDK 25
      JDK_DIR="/usr/lib/jvm/java-25-openjdk-amd64"                                                          # Localização da JDK

      wget -q -O /tmp/jdk25.tar.gz "${JDK_URL}"                                                             # Baixa a JDK da URL
      mkdir -p "${JDK_DIR}"                                                                                 # Cria um diretório para a JDK
      tar -xzf /tmp/jdk25.tar.gz -C "${JDK_DIR}" --strip-components=1                                       # Descompacta o tar para o endereço definido
      rm /tmp/jdk25.tar.gz                                                                                  

      # Adiciona a JDK 25 aos pacotes no sistema
      update-alternatives --install /usr/bin/java java "${JDK_DIR}/bin/java" 100
      update-alternatives --install /usr/bin/javac javac "${JDK_DIR}/bin/javac" 100
      update-alternatives --set java "${JDK_DIR}/bin/java"
      update-alternatives --set javac "${JDK_DIR}/bin/javac"

      echo 'JAVA_HOME="/usr/lib/jvm/java-25-openjdk-amd64"'        >> /etc/environment                     
      echo 'export JAVA_HOME="/usr/lib/jvm/java-25-openjdk-amd64"' >> /etc/profile.d/java.sh
      echo 'export PATH=$JAVA_HOME/bin:$PATH'                       >> /etc/profile.d/java.sh              
      export JAVA_HOME="/usr/lib/jvm/java-25-openjdk-amd64"                                                # Define a variável de ambiente JAVA_HOME
      export PATH=$JAVA_HOME/bin:$PATH                                                                     # Adiciona a variável ao PATH

      mvn clean package -DskipTests --file /vagrant/app/user-auth-api/pom.xml                              # Compila a aplicação Java Spring

      # Baixa o servidor Tomcat 11
      sudo wget -O tomcat.tar.gz https://dlcdn.apache.org/tomcat/tomcat-11/v11.0.22/bin/apache-tomcat-11.0.22.tar.gz
      sudo mkdir /etc/tomcat11
      sudo tar -xzf tomcat.tar.gz -C /etc/tomcat11 --strip-components=1
      sudo useradd -r -m -U -d /etc/tomcat11 -s /bin/false tomcat
      sudo chown -R tomcat: /etc/tomcat11
      sudo sh -c 'chmod +x /etc/tomcat11/bin/*.sh'


      # Define as credenciais de acesso ao servidor
      cat > /etc/tomcat11/conf/tomcat-users.xml <<EOF
<?xml version='1.0' encoding='utf-8'?>
<tomcat-users xmlns="http://tomcat.apache.org/xml"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
              version="1.0">

  <role rolename="manager-gui"/>
  <role rolename="manager-script"/>
  <role rolename="admin-gui"/>

  <user username="admin"
        password="admin123"
        roles="manager-gui,manager-script,admin-gui"/>
</tomcat-users>
EOF

      # Permitir acesso ao servidor fora do localhsot
      cat > /etc/tomcat11/webapps/manager/META-INF/context.xml <<EOF
<Context antiResourceLocking="false" privileged="true" ignoreAnnotations="true">
  <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                   sameSiteCookies="strict" />
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>
EOF

      # Permitir acesso ao servidor fora do localhsot
      cat > /etc/tomcat11/webapps/host-manager/META-INF/context.xml <<EOF
<Context antiResourceLocking="false" privileged="true" ignoreAnnotations="true">
  <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                   sameSiteCookies="strict" />
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>
EOF

      # Cria o serviço Tomcat
      cat > /etc/systemd/system/tomcat.service <<EOF
[Unit]
Description=Apache Tomcat 11 Web Application Server
After=network.target

[Service]
Type=forking  
User=tomcat
Group=tomcat
Environment="JAVA_HOME=/usr/lib/jvm/java-25-openjdk-amd64"
Environment="CATALINA_HOME=/etc/tomcat11"
Environment="CATALINA_BASE=/etc/tomcat11"
Environment="CATALINA_PID=/etc/tomcat11/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"
ExecStart=/etc/tomcat11/bin/startup.sh
ExecStop=/etc/tomcat11/bin/shutdown.sh
Environment="DB_HOST=192.168.56.11"                                                  
Environment="DB_USER=postgres"
Environment="DB_PASSWORD=12345"

[Install]
WantedBy=multi-user.target
EOF

      systemctl daemon-reload                          # Recarrega todos os serviços
      systemctl enable tomcat                          # Habilita o servidor
      systemctl start tomcat                           # Inicializa o servidor

      java -version
      systemctl status tomcat --no-pager

      # Adiciona a aplicação ao contexto de deploy
      cp /vagrant/app/user-auth-api/target/*.war /etc/tomcat11/webapps
    SHELL
  end
end