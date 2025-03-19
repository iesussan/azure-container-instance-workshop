## Menú
- [NGroups (Preview)](#ngroups-preview)
    - [Arquitectura de Alto Nivel y Conceptos Clave](#arquitectura-de-alto-nivel-y-conceptos-clave)

## NGroups (Preview)

## Arquitectura de Alto Nivel y Conceptos Clave

NGroups provee funcionalidades avanzadas para manejar multiples capacidades sobre los Container Groups.
De los elementos principales se incluyen:
- Es capaz de manater multiples instancias
- Permite Rolling Updates.
- Permite la implementacion de alta disponibilidad gracias al soporte de Availability Zones (AZs)
- Soporta Managed Identity
- Soporta Confidential container
- Soporta la implementacion de Balanceadores.
- Soporta Zone rebalancing (Zone Any)

## Grafico de Aquitectura de alto nivel

![Container Group Example](./images/ngroups-overview.png)

La imagen ilustra la diferencia entre:
1.	**Crear contenedores individualmente en ACI:**
    - En la parte izquierda se ve cómo el usuario realiza llamadas directas a la API de Azure Container Instances (ACI) para crear varios grupos de contenedores (CG1, CG2, CG3).
    - Cada contenedor o grupo de contenedores se gestiona de manera independiente, y el usuario debe realizar varias llamadas (PUT CG1, PUT CG2, PUT CG3) para crearlos o actualizarlos.
2.	Usar NGroups para gestionar varios contenedores de forma centralizada:
    - En la parte derecha se ve un único recurso llamado “NGroups: Foo”, con un Desired count: 3 y una referencia a un “CGProfile” (Container Group Profile).
    - NGroups actúa como una capa adicional sobre ACI. El usuario hace una sola llamada (PUT NGroups: Foo) indicando cuántos grupos de contenedores (N) quiere y con qué propiedades (definidas en el CGProfile).
    - NGroups se encarga de llamar internamente a las APIs de ACI (PUT CGX) para crear y mantener esos contenedores.

3. En la sección inferior derecha se muestra cómo NGroups administra automáticamente la disponibilidad y la escala de los grupos de contenedores:
    - El recurso NGroups “Foo” tiene 3 instancias en ejecución: Foo_1, Foo_2 y Foo_3.
    - Cuando una de las instancias falla (Failed Instance Foo_3), NGroups detecta la caída y crea una nueva instancia (Foo_4) para mantener el número deseado de contenedores (Desired count = 3).

En resumen:
    - Sin NGroups, el usuario gestiona cada Container Group de manera individual llamando a la API de ACI tantas veces como contenedores requiera.
    - Con NGroups, se especifica un perfil (CGProfile) y un número deseado de instancias (N). NGroups se encarga de crear y mantener esos N contenedores, lo que facilita tareas como la escala, las actualizaciones y la resiliencia frente a fallos.

NGroups se basa en dos componentes principales:

1. **Container Group Profile (CGProfile):**  
   Actúa como una plantilla que define las propiedades comunes de los Container Groups, como la imagen del contenedor, recursos (CPU, memoria), restart policy, configuración de IP y otros parámetros. Esto evita la duplicación de configuraciones y reduce la sobrecarga de administración cuando se gestionan múltiples CGs.

2. **NGroups Resource:**  
   Una vez creado el CGProfile, se crea un recurso NGroups que referencia dicho perfil. Al definir el número deseado de instancias (desiredCount) y otras propiedades (como zonas, identidades o balanceadores), NGroups invoca las API de ACI para crear o actualizar los Container Groups conforme a la plantilla definida en el CGProfile.

### Beneficios de Usar NGroups

- **Consistencia y eficiencia:** Al centralizar la configuración en un CGProfile, se garantiza que todos los CGs tengan propiedades consistentes.
- **Escalabilidad simplificada:** Se pueden crear y administrar 'n' Container Groups con una sola operación de API.
- **Flexibilidad en actualizaciones:** Permite actualizaciones manuales o progresivas (rolling), minimizando el impacto en la producción.
- **Distribución en múltiples zonas:** Asegura que la aplicación siga operativa incluso si una zona falla, ya que los CGs se distribuyen en varias Availability Zones.


### Ejemplo de ARM Template para NGroups

A continuación se muestra un ejemplo de ARM Template que crea un CGProfile y un recurso NGroups que distribuye los Container Groups en zonas múltiples. Este template ilustra cómo se configura el desiredCount, se referencia el CGProfile y se establecen las zonas de disponibilidad.

1.	Registrar la característica de NGroupsPreview en tu suscripción:

```bash
az feature register --name NGroupsPreview --namespace Microsoft.ContainerInstance
```
2.	Revisar el estado de la característica:
```bash
az feature show --name NGroupsPreview --namespace Microsoft.ContainerInstance
```
    nota: Espera hasta que el estado sea Registered.

3.	Registrar el proveedor de recursos de ACI nuevamente para que tome efecto la nueva característica:
```
az provider register --namespace Microsoft.ContainerInstance
```

### Plantilla ARM para NGroups

A diferencia de un container group tradicional, al usar NGroups primero creamos un Container Group Profile y luego un NGroups que hace referencia a ese perfil. Este ejemplo ARM template contiene ambos recursos:

1.	Container Group Profile (tipo Microsoft.ContainerInstance/containerGroupProfiles)
2.	NGroups (tipo Microsoft.ContainerInstance/NGroups)

Crea un archivo llamado ngroups-deployment.json con el siguiente contenido (ajusta parámetros según necesites):

```bash
touch ngroups-deployment.json
code ngroups-deployment.json
```
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "location": {
      "type": "string",
      "defaultValue": "brazilsouth"
    },
    "cgProfileName": {
      "type": "string",
      "defaultValue": "myCGProfile"
    },
    "nGroupsName": {
      "type": "string",
      "defaultValue": "myNGroup"
    },
    "desiredCount": {
      "type": "int",
      "defaultValue": 3
    },
    "zonesArray": {
      "type": "array",
      "defaultValue": [
        "1",
        "2",
        "3"
      ],
      "metadata": {
        "description": "Zonas de disponibilidad para distribuir los contenedores."
      }
    }
  },
  "variables": {
    "resourcePrefix": "[concat('/subscriptions/', subscription().subscriptionId, '/resourceGroups/', resourceGroup().name, '/providers/')]"
  },
  "resources": [
    {
      "apiVersion": "2024-11-01-preview",
      "type": "Microsoft.ContainerInstance/containerGroupProfiles",
      "name": "[parameters('cgProfileName')]",
      "location": "[parameters('location')]",
      "properties": {
        "sku": "Standard",
        "containers": [
          {
            "name": "aci-tutorial-app",
            "properties": {
              "image": "mcr.microsoft.com/azuredocs/aci-helloworld:latest",
              "ports": [
                {
                  "protocol": "TCP",
                  "port": 80
                }
              ],
              "resources": {
                "requests": {
                  "memoryInGB": 1.5,
                  "cpu": 1
                }
              }
            }
          }
        ],
        "restartPolicy": "OnFailure",
        "ipAddress": {
          "type": "Public",
          "ports": [
            {
              "protocol": "TCP",
              "port": 80
            }
          ]
        },
        "osType": "Linux"
      }
    },
    {
      "apiVersion": "2024-11-01-preview",
      "type": "Microsoft.ContainerInstance/NGroups",
      "name": "[parameters('nGroupsName')]",
      "location": "[parameters('location')]",
      "dependsOn": [
        "[resourceId('Microsoft.ContainerInstance/containerGroupProfiles', parameters('cgProfileName'))]"
      ],
      "properties": {
        "elasticProfile": {
          "desiredCount": "[parameters('desiredCount')]",
          "maintainDesiredCount": true
        },
        "updateProfile": { 
          "updateMode": "Rolling",
          "rollingUpdateProfile": { 
            "inPlaceUpdate": true 
          } 
        },
        "containerGroupProfiles": [
          {
            "resource": {
              "id": "[resourceId('Microsoft.ContainerInstance/containerGroupProfiles', parameters('cgProfileName'))]"
            }
          }
        ]
      },
      "zones": "[parameters('zonesArray')]"
    }
  ],
  "outputs": {
    "nGroupsResourceId": {
      "type": "string",
      "value": "[resourceId('Microsoft.ContainerInstance/NGroups', parameters('nGroupsName'))]"
    }
  }
}
```
## Desplegar plantilla ARM para Ngroups
1.	Crear o seleccionar un grupo de recursos (si no existe):

    ```bash
    az group create --name myResourceGroup --location brazilsouth --tags environment=lab app="Workshop ACI" owner-ops="tec-it operations & infrastructure" owner-dev="tec-it integration & application dev"
    ```

2.	Desplegar la plantilla que contiene containerGroupProfiles y NGroups:

    ```bash
    az deployment group create \
    --resource-group myResourceGroup \
    --template-file ngroups-deployment.json \
    --parameters location=brazilsouth cgProfileName=myCGProfile nGroupsName=myNGroup desiredCount=3
    ```

3.	Verificar que se haya creado el recurso:

    ```bash
    az resource show \
    --resource-group myResourceGroup \
    --name myNGroup \
    --resource-type Microsoft.ContainerInstance/NGroups \
    --query properties \
    --output json
    ```
4. Escalar o Actualizar NGroups

Para cambiar el número de instancias en tu NGroups, puedes actualizar el desiredCount:

```bash
    az deployment group create \
    --resource-group myResourceGroup \
    --template-file ngroups-deployment.json \
    --parameters location=brazilsouth cgProfileName=myCGProfile nGroupsName=myNGroup desiredCount=5
```