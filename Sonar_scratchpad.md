This document consolidates the entire installation and troubleshooting workflow for SonarQube v10.4+ on Ubuntu 26.04/22.04 with a PostgreSQL 18 backend.

SonarQube Enterprise Deployment: Troubleshooting & Configuration
This guide addresses common deployment failures when running SonarQube with Java 21 and PostgreSQL 18.

1. Architectural Overview
SonarQube operates as three distinct processes: the Web Server, the Compute Engine, and the Elasticsearch search engine. Each must be correctly configured to interface with the database and handle memory/security constraints.

2. Issue Resolution Roadmap
A. Java 21 Security Manager Bypass
Modern Java versions have deprecated System.setSecurityManager(). Without explicit overrides, the Web Server and Compute Engine will crash on startup.

Resolution:
Add the following to ~/sonarqube-10.4.1.88267/conf/sonar.properties:

Properties
sonar.web.javaAdditionalOpts=-Djava.security.manager=allow
sonar.ce.javaAdditionalOpts=-Djava.security.manager=allow
B. PostgreSQL Deployment & User Management
If the postgres user is missing or the package is in a "ghost state" (installed but non-functional), it must be purged and re-initialized.

Resolution:

Purge broken metadata:

Bash
sudo apt-get purge -y postgresql-18 postgresql-client-18 postgresql-common
sudo apt-get autoremove -y
Force fresh installation:

Bash
sudo apt-get install -y postgresql-18 postgresql-client-18
C. Database Provisioning & Schema Ownership
SonarQube requires a dedicated user and schema to prevent H2 database corruption.

Resolution:

Bash
sudo -i -u postgres psql -c "CREATE USER sonar WITH PASSWORD 'admin';"
sudo -i -u postgres psql -c "CREATE DATABASE sonarqube OWNER sonar;"
sudo -i -u postgres psql -d sonarqube -c "ALTER SCHEMA public OWNER TO sonar; GRANT ALL PRIVILEGES ON SCHEMA public TO sonar;"
3. Production Configuration Template
Append these settings to ~/sonarqube-10.4.1.88267/conf/sonar.properties:

Properties
# Database Connectivity
sonar.jdbc.username=sonar
sonar.jdbc.password=admin
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube

# Java 21 Bypass Flags
sonar.web.javaAdditionalOpts=-Djava.security.manager=allow
sonar.ce.javaAdditionalOpts=-Djava.security.manager=allow
4. Operational Recovery (Hard Reset)
If processes collide due to dangling PIDs or corrupted temp files:

Terminate all processes: pkill -u sonarqube -f java

Flush volatile state: rm -rf ~/sonarqube-10.4.1.88267/temp/*

Remove lock files: rm -f ~/sonarqube-10.4.1.88267/bin/linux-x86-64/SonarQube.pid

Initialize daemon: ~/sonarqube-10.4.1.88267/bin/linux-x86-64/sonar.sh start

5. Verification
Run the following to confirm the system state:

Service Status: ./sonar.sh status

Network Listener: curl -I http://localhost:9000

Process Monitor: ps aux | grep sonar

Note: If the web UI is accessible but status shows "not running," ensure the process was launched via sonar.sh start (daemon mode) rather than sonar.sh console (interactive mode).

Documentation compiled for SonarQube version 10.4.1.88267.