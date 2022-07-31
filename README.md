# SonarQube Setup for SPA(Reactjs) & API(.net Core) through Azure Build Pipeline
## Use case:
1.	Setup the SonarQube for code analysis
2.	Create a Build Pipeline (CI) for both .net Core and React Application using YAML
3.	Configure the SonarQube code analysis for both .net Core and React

## Solution
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

4. The default port for SonarQube is 9000. l SonarQube URL will be http://<YOUR_PUBLIC_IP_ADDRESS>:9000

5. Login with Default username(admin) and Passcode(admin) 

6. Click on the Create New project and select Azure pipeline and follow the steps given there.
