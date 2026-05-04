Vagrant.configure("2") do |config|
  config.vm.define :db do |db_config|
    db_config.vm.box = "bento/ubuntu-24.04"
    db_config.vm.hostname = "postgresql-server"
    db_config.vm.network :private_network, ip: "192.168.56.11"

    db_config.vm.synced_folder ".", "/vagrant", type: "virtualbox"
    
    db_config.vm.provider "virtualbox" do |vb|
      vb.name   = "postgresql-server"
      vb.memory = "2048"
      vb.cpus   = 2
    end

    db_config.vm.provision "shell", inline: <<-SHELL
      sudo apt-get update -y
      sudo apt-get upgrade -y
      sudo apt-get install -y postgresql
      sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" /etc/postgresql/*/main/postgresql.conf

      sudo cat >> /etc/postgresql/*/main/pg_hba.conf <<EOF
host    all             all             0.0.0.0/0               scram-sha-256
host    all             all             ::/0                    scram-sha-256
EOF

      DB_NAME="db_auth"
      DB_ROOT_PASS="12345"

      sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD '${DB_ROOT_PASS}';"
      sudo -u postgres psql -c "CREATE DATABASE ${DB_NAME};"
      sudo systemctl restart postgresql
      sudo -u postgres psql -f /vagrant/sources/init.sql
    SHELL
  end

Vagrant.configure("2") do |config| 
  config.vm.define :web do |web_config|
    web_config.vm.box = "bento/ubuntu-25.04"
    web_config.vm.hostname = "tomcat-server"
    web_config.vm.network :private_network, ip: "192.168.56.10"
    web_config.vm.network "forwarded_port", guest: 8080, host: 8080   
    web_config.vm.network "forwarded_port", guest: 8443, host: 8443   

    web_config.vm.synced_folder ".", "/vagrant", type: "virtualbox"

    web_config.vm.provider "virtualbox" do |vb|
      vb.name   = "tomcat-server2"
      vb.memory = "4096"
      vb.cpus   = 2
    end

    web_config.vm.provision "shell", inline: <<-SHELL
      apt-get update -y
      apt-get upgrade -y

      apt-get install -y openjdk-25-jdk
      apt-get install maven

      echo 'JAVA_HOME="/usr/lib/jvm/java-25-openjdk-amd64"'       >> /etc/environment
      echo 'export JAVA_HOME="/usr/lib/jvm/java-25-openjdk-amd64"' >> /etc/profile.d/java.sh
      echo 'export PATH=$JAVA_HOME/bin:$PATH'                       >> /etc/profile.d/java.sh
      source /etc/environment

      mvn clean package -DskipTests --file /vagrant/app/user-auth-api/pom.xml

      sudo wget -O tomcat.tar.gz https://dlcdn.apache.org/tomcat/tomcat-11/v11.0.21/bin/apache-tomcat-11.0.21.tar.gz
      sudo mkdir /etc/tomcat11
      sudo tar -xzf tomcat.tar.gz -C /etc/tomcat11 --strip-components=1
      sudo useradd -r -m -U -d /etc/tomcat11 -s /bin/false tomcat
      sudo chown -R tomcat: /etc/tomcat11
      sudo sh -c 'chmod +x /etc/tomcat11/bin/*.sh'

    sudo cat > /etc/tomcat11/conf/tomcat-users.xml <<EOF
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

    sudo cat > /etc/tomcat11/webapps/manager/META-INF/context.xml <<EOF
<Context antiResourceLocking="false" privileged="true" ignoreAnnotations="true">
  <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                   sameSiteCookies="strict" />
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>
EOF

    sudo cat >  /etc/tomcat11/webapps/host-manager/META-INF/context.xml <<EOF
<Context antiResourceLocking="false" privileged="true" ignoreAnnotations="true">
  <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                   sameSiteCookies="strict" />
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>
EOF

    sudo touch /etc/systemd/system/tomcat.service
    sudo cat > /etc/systemd/system/tomcat.service <<EOF
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

[Install]
WantedBy=multi-user.target
EOF

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

    sudo systemctl daemon-reload
    sudo systemctl enable tomcat
    sudo systemctl start tomcat

    java -version
    sudo systemctl status tomcat --no-pager
    
    sudo cp /vagrant/app/user-auth-api/target/*.war /etc/tomcat11/webapps
  SHELL
  end
end
