# SonarQube Setup for SPA(ReactJS) & API(.Net 5.0) through Azure Build Pipeline
## Use case:
1.	Setup the SonarQube for code analysis
2.	Create a Build Pipeline (CI) for both .net Core and React Application using YAML
3.	Configure the SonarQube code analysis for both .net 5.0  and React

## Solution 1 (Using Container Instance)
<p align="center" width="100%">
    <img width="50%" src="https://github.com/thavasnippets/sonarqube-az-devops/blob/main/Sonarqube-AZ.png">
</p>

### SonarQube Setup:
1. Launch the Azure Cloud Shell
2. Create Resource group   
```bash
az group create --name rg-sonarqube --location eastus
```
3. Create Azure Container Instance with SonarQube Docker image

```bash
az container create -g rg-sonarqube \
--name sonarqubeserver \
--image sonarqube \
--cpu 2  \
--memory 3.5  \
--ip-address Public  \
--protocol TCP \
--ports 9000  \
```
4. The default port for SonarQube is 9000. SonarQube URL will be http://<YOUR_PUBLIC_IP_ADDRESS>:9000

5. Login with Default username(admin) and Passcode(admin) 

6. Click on the Create New project and select Azure pipeline and follow the steps given there.

## Solution 2 (Using Virtual Machine)
<p align="center" width="100%">
    <img width="50%" src="https://github.com/thavasnippets/sonarqube-az-devops/blob/ab1e8d86d9b305e0741fcc8c528381537cc94818/Sonarqube-AZVM.png">
</p>

1. Launch the Azure Cloud Shell
2. Create Resource group   
```bash
az group create --name rg-sonarqube-vm --location eastus
```
3. Create Azure Virtual machine with linux(Ubuntu) image
```
az vm create \
  --resource-group rg-sonarqube-vm \
  --name vmsonarqube \
  --image UbuntuLTS \
  --admin-username <<USERNAME>> \
  --admin-password <<PASSWORD>> \
  --public-ip-address sonarPubIp \
  --authentication-type password
```
4. Open 9000 port in VM
```bash
az vm open-port -g rg-sonarqube-vm -n vmsonarqube --port 9000 --priority 100
```
5. Run the below script using Runshellscript Module to do the below activities
    1. Install and Create a docker compose file in the VM to run the sonarqube image    
```bash
az vm run-command invoke -g rg-sonarqube-vm -n vmsonarqube --command-id RunShellScript --scripts '
echo "vm.max_map_count=262144" | sudo tee -a  /etc/sysctl.conf 
echo "fs.file-max=65536" | sudo tee -a  /etc/sysctl.conf
sudo apt-get update
sudo apt-get install docker-compose -y
sudo usermod -aG docker $USER
echo "
version: \"3\"
services:
  sonarqube:
    image: sonarqube:lts-community
    container_name: sonarqube
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - sonarqube_conf:/opt/sonarqube/conf
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_bundled-plugins:/opt/sonarqube/lib/bundled-plugins
volumes:
  sonarqube_bundled-plugins:
  sonarqube_conf:
  sonarqube_data:
  sonarqube_db:
  sonarqube_extensions:
" | sudo tee /etc/docker-compose.yml
cd /etc/
sudo mv docker-compose.yml /
cd /
sudo sysctl -w vm.max_map_count=262144
sudo docker-compose up -d
sudo docker-compose logs --follow' 
```
6. The default port for SonarQube is 9000. SonarQube URL will be http://<YOUR_PUBLIC_IP_ADDRESS>:9000

7. Login with Default username(admin) and Passcode(admin) 

8. Click on the Create New project and select Azure pipeline and follow the steps given there.

## Move the Data to external database
In the above 2 solutions the data is getting saved in the local (i.e inside the container or VM  instance) if the container restarts or the Sonarqube crashes we will  endup with configuration and data losses. To avoid we can move the data to external database using JDBC Configuration 

<p align="center" width="100%">
    <img width="50%" src="https://github.com/thavasnippets/sonarqube-az-devops/blob/ab1e8d86d9b305e0741fcc8c528381537cc94818/Sonarqube-AZ-VM-DB.png">
</p>

### Assumption:
SQL Server is already created

### JDBC Configuration
```yaml
version: "3"
services:
  sonarqube:
    image: sonarqube:lts-community
    container_name: sonarqube
    restart: unless-stopped
    environment:
      - SONARQUBE_JDBC_USERNAME=<<JDBC_USERNAME>>
      - SONARQUBE_JDBC_PASSWORD=<<JDBC_PASSWORD>>
      - SONARQUBE_JDBC_URL=jdbc:sqlserver://<<SQLSERVER_NAME>>.database.windows.net:1433;database=<<DATABASE_NAME>>;user=<<USERNAME>>@<<SERVER_NAME>>;password=<<PASSWORD>>;encrypt=true;trustServerCertificate=false;hostNameInCertificate=*.database.windows.net;loginTimeout=30
    ports:
      - "9000:9000"
      - "9092:9092"
    volumes:
      - sonarqube_conf:/opt/sonarqube/conf
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_bundled-plugins:/opt/sonarqube/lib/bundled-plugins
volumes:
  sonarqube_bundled-plugins:
  sonarqube_conf:
  sonarqube_data:
  sonarqube_db:
  sonarqube_extensions:
```

## Azure Build Pipeline Setup:

### .NET Web API:

1. Prepare Analysis Configuration (Before Build)
```yaml
- task: SonarQubePrepare@5
      inputs:
        SonarQube: 'sonar' #Service Connection Name
        scannerMode: 'MSBuild'
        projectKey: '<<PROJECT KEY FROM SONARQUBE>>' 
```
2. Run Code Analysis (After Build)
```yaml
 - task: SonarQubeAnalyze@5
```
3. Publish Code Analysis
```yaml
- task: SonarQubePublish@5
```

### Reactjs UI:-
1. Prepare Analysis Configuration (Before Build)
```yaml
- task: SonarQubePrepare@5
      inputs:
        SonarQube: sonar  #Service Connection Name
        scannerMode: CLI
        configMode: manual
        cliProjectKey: '<<SONARQUBE PROJECT KEY>>'
        cliProjectName: <<SONARQUBE PROJECT NAME>>
        cliSources: 'ui'
```
2. Run Code Analysis (After Build)
```yaml
 - task: SonarQubeAnalyze@5
```
3. Publish Code Analysis
```yaml
- task: SonarQubePublish@5
```

#### Its Completed!

### Happy Coding!
