# Enumeración con CMD y PowerShell

| Comando          | Función              |
| ---------------- | -------------------- |
| net user         | Usuarios locales     |
| whoami           | Usuario actual       |
| whoami /groups   | Grupos del usuario   |
| net user /domain | Usuarios del dominio |
| net user /domain | Info detallada       |

#Get-DomainComputer se utiliza para enumerar computadores unidos a un dominio Active Directory, 
filtros 
#Get-DomainComputer | Select Name, Description | Sort Name.
#Get-DomainComputer -OperatingSystem "*Server*"
Para validar si el equipo esta online 
#Get-DomainComputer -Ping

#Get-DomainComputer -Unconstrained
Enumera equipos del dominio donde el atributo:

```
TRUSTED_FOR_DELEGATION
```
está habilitado.
En Active Directory, esto significa que el equipo puede recibir y reutilizar tickets Kerberos delegados de usuarios que se autentican contra él.

```
Get-DomainSID
```
¿Qué hace?
Obtiene el **Security Identifier (SID)** único del dominio actual.
El SID identifica de forma única al dominio dentro de Active Directory.

```
(Get-DomainPolicy)."SystemAccess"
``` 
¿Qué hace?
Extrae específicamente la sección **SystemAccess** de la política del dominio.
Esta sección contiene configuraciones relacionadas con:

- Contraseñas
- Bloqueo de cuentas
- Restricciones de acceso
- Políticas de autenticación

(Get-DomainPolicy -domain <Domain>)."systemaccess"
Este comando consulta la política del dominio especificado y extrae únicamente la sección SystemAccess

Obtiene la política de Kerberos
(Get-DomainPolicy)."KerberosPolicy"

Obtiene información sobre los **controladores de dominio** disponibles en Active Directory.
Get-NetDomainController -Domain <Domain>

Obtiene información del DC Primario
Get-NetDomain | Select-Object 'PdcRoleOwner'

Enumerar todos los DC en el Bosque
Get-NetForestDomain

Enumera todos los DC de confianza en el bosque
Get-NetForestDomain -Verbose | Get-DomainTrust

Enumera DC dentro del bosque con relaciones externas de confianza
Get-NetForestDomain -Verbose | Get-DomainTrust | ?{$_.TrustType -eq 'External'}

Información del Boque Actual
Get-NetForest

Lista los DC del bosque actual
Get-NetForestDomain

Enumera un Bosque Especifico
Get-NetForest -Forest <Forest>

Lista todos los Grupos del dominio
Get-NetGroup o Get-NetGroup -Properties SamAccountName | Sort SamAccountName

Lista los Grupos de un usuario
Get-DomainGroup -MemberIdentity "<User>" 

Lista todos lo grupos con admin 
Get-NetGroup "*admin*" -Properties SamAccountName | Sort SamAccountName

Lista los grupo de un DC
Get-NetGroup -Domain <Domain>

Enumerar los DC y los Grupos Locales
Get-NetDomainController | Get-NetLocalGroup

Lista los grupos locales de un computador
Get-NetLocalGroup -ComputerName <Hostname>

Lista gMSA
Get-DomainObject -LDAPFilter '(objectClass=msDS-GroupManagedServiceAccount)'

Ve quien puede recupera la contraseña de gMSA
Get-ADServiceAccount -Identity [Identity] -Properties * | Select-Object PrincipalsAllowedToRetrieveManagedPassword

Recuperar la contraseña de un gMSA
.\GMSAPasswordReader.exe --accountname [GMSA-Account]

|Comando|Qué hace|
|---|---|
|`Get-DomainObject -LDAPFilter '(objectClass=msDS-GroupManagedServiceAccount)'`|Enumera gMSA del dominio|
|`Get-ADServiceAccount -Identity [Identity] -Properties * \| Select-Object PrincipalsAllowedToRetrieveManagedPassword`|Muestra quién puede recuperar la contraseña de una gMSA|
|`.\GMSAPasswordReader.exe --accountname [GMSA-Account]`|Intenta recuperar la contraseña de una gMSA autorizada|
|`Get-DomainGPO -Properties DisplayName, CN`|Lista GPO del dominio|
|`Get-DomainGPO -ADSPath ((Get-NetOU "StudentMachines" -FullData).gplink.split(";")[0] -replace "^.")`|Muestra GPO aplicada a una OU específica|
|Script con `Get-DomainOU`|Enumera OUs y GPO aplicadas a cada una|

# Resumen rápido - GPO Enumeration

| Comando | Qué hace |
|---|---|
| `Get-DomainGPO -ComputerIdentity <FQDN>` | Obtiene las GPO aplicadas a un computador específico usando su FQDN. |
| `Get-DomainGPO -ComputerIdentity <FQDN> \| Select DisplayName,CN` | Muestra solo el nombre visible y CN de las GPO aplicadas a un computador. |
| `Get-DomainGPO -UserIdentity <SamAccountName>` | Obtiene las GPO aplicadas a un usuario específico usando su SAM Account Name. |
| `Get-DomainGPO -UserIdentity <SamAccountName> \| Select DisplayName,CN` | Muestra solo el nombre visible y CN de las GPO aplicadas a un usuario. |
| `Get-NetGPOGroup` | Lista grupos configurados mediante GPO, especialmente grupos restringidos o con permisos definidos por política. |
| `Get-NetGPOGroup -ResolveMembersToSIDs` | Lista grupos configurados mediante GPO y resuelve sus miembros a SIDs. |
| `$GroupNames = Get-NetGPOGroup -ResolveMembersToSIDs \| Select-Object -ExpandProperty "GroupName"; foreach ($GroupName in $GroupNames) { $ModifiedGroupName = $GroupName -replace '^.*\\'; Get-DomainGroupMember -Identity $ModifiedGroupName }` | Obtiene los grupos definidos en GPO y enumera los miembros de cada grupo. |
| `Find-GPOComputerAdmin -ComputerName <FQDN>` | Busca GPOs que otorgan permisos administrativos locales sobre un computador específico. |

## Idea clave

- `Get-DomainGPO -ComputerIdentity` → identifica políticas aplicadas a un equipo.
- `Get-DomainGPO -UserIdentity` → identifica políticas aplicadas a un usuario.
- `Get-NetGPOGroup` → enumera grupos administrados o restringidos por GPO.
- `Find-GPOComputerAdmin` → ayuda a descubrir quién puede ser administrador local en una máquina por efecto de una GPO.

# Resumen rápido - OU y GPO Location Enumeration

| Comando | Qué hace |
|---|---|
| `Find-GPOLocation -ComputerName <FQDN>` | Identifica las GPO que se aplican a un computador específico y ayuda a determinar qué políticas afectan a esa máquina. |
| `Get-DomainOU` | Enumera todas las Organizational Units (OU) del dominio. |
| `Get-DomainOU "*OU name*"` | Busca información de una OU específica que coincida con el patrón indicado. |

## Idea clave

- `Find-GPOLocation` → permite correlacionar una máquina con las GPO que le aplican.
- `Get-DomainOU` → permite ver la estructura organizacional del dominio.
- `Get-DomainOU "*OU name*"` → permite buscar una OU concreta por nombre.

## Uso en pentesting

Estos comandos ayudan a identificar:

- Qué políticas afectan a una máquina.
- Qué OUs existen en el dominio.
- Qué equipos o usuarios podrían estar agrupados bajo una OU específica.
- Posibles rutas de escalada mediante GPO mal configuradas.

# Get-DomainUser -SPN

| Comando | Qué hace |
|---|---|
| `Get-DomainUser -SPN` | Enumera usuarios del dominio que tienen un Service Principal Name (SPN) configurado. |
| `Get-DomainUser -SPN \| Select-Object Name, ServicePrincipalName` | Muestra solo el nombre del usuario y sus SPN asociados. |

## ¿Para qué sirve?

Este comando se usa para identificar **cuentas de servicio** dentro de Active Directory.

Las cuentas con SPN son importantes porque pueden ser objetivo de **Kerberoasting**.

## Campo importante

| Campo | Significado |
|---|---|
| `Name` | Nombre del usuario o cuenta de servicio. |
| `ServicePrincipalName` | Servicio asociado a la cuenta, por ejemplo MSSQL, HTTP, CIFS, etc. |

## Ejemplo de salida

```text
Name        ServicePrincipalName
----        --------------------
svc_sql     MSSQLSvc/sql01.domain.local:1433
svc_web     HTTP/web01.domain.local


# Resumen rápido - User Description y GPO Takeover Enumeration

| Comando | Qué hace |
|---|---|
| `Get-DomainUser -Properties samaccountname,description \| Where-Object {$_.description -ne $null}` | Enumera usuarios del dominio que tienen contenido en el campo `description`. |
| `Get-DomainGPO \| Get-DomainObjectAcl -ResolveGUIDs \| Where-Object { $_.ActiveDirectoryRights -match "CreateChild\|WriteProperty" -and $_.SecurityIdentifier -match "S-1-5-21-569305411-121244042-2357301523-[\d]{4,10}" }` | Busca GPOs donde algún principal del dominio tenga permisos potencialmente peligrosos como `CreateChild` o `WriteProperty`. |
| `Get-DomainGPO -Identity "CN={5059FAC1-5E94-4361-95D3-3BB235A23928},CN=Policies,CN=System,DC=dev,DC=cyberbotic,DC=io" \| Select-Object displayName, gpcFileSysPath` | Obtiene el nombre visible y la ruta SYSVOL de una GPO específica usando su Distinguished Name. |
| `ConvertFrom-SID <SID>` | Convierte un SID a su usuario, grupo o computador correspondiente. |
| `ConvertFrom-SID S-1-5-21-569305411-121244042-2357301523-1105` | Resuelve un SID específico para identificar qué principal tiene permisos sobre la GPO. |

## Explicación breve

### `Get-DomainUser -Properties samaccountname,description | Where-Object {$_.description -ne $null}`

Busca usuarios con información en el campo `description`.

Esto es útil porque a veces administradores dejan notas sensibles como:

- Contraseñas temporales
- Nombre del servicio
- Información del cargo
- Comentarios administrativos

---

### `Get-DomainGPO | Get-DomainObjectAcl -ResolveGUIDs | Where-Object {...}`

Busca permisos peligrosos sobre GPOs.

Permisos como:

- `CreateChild`
- `WriteProperty`

pueden indicar que un usuario o grupo podría modificar una GPO.

> Si un usuario puede modificar una GPO que se aplica a equipos o usuarios privilegiados, puede ser una ruta de escalada.

---

### `Get-DomainGPO -Identity "<DN>" | Select-Object displayName, gpcFileSysPath`

Obtiene detalles de una GPO específica.

Campos importantes:

| Campo | Significado |
|---|---|
| `displayName` | Nombre visible de la GPO |
| `gpcFileSysPath` | Ruta en SYSVOL donde está almacenada la política |

---

### `ConvertFrom-SID <SID>`

Convierte un SID a un nombre legible.

Ejemplo:

```powershell
ConvertFrom-SID S-1-5-21-569305411-121244042-2357301523-1105
# LDAPSearch



ldapsearch -x -H ldap://IP -b ""
Parámetros clave:
- `-x` → autenticación simple
- `-h` → IP del DC
- `-b` → base del dominio
- `-D` → usuario
- `-w` → contraseña
- `-s` → scope (base, one, sub)

Siempre primero buscar el contexto del dominio para poner en -b
EJ: defaultNamingContext: DC=empresa,DC=local

Realizar enumeración de LDAP
nmap -p 389,636 -sV <DC-IP>
Buscar RootDSE(Sin Credenciales) -b
ldapsearch -LLL -x -H ldap://<DC-IP> -b "" -s base "(objectclass=*)"
	defaultNamingContext: DC=empresa,DC=local
Intentar Realizar Autenticación Anonima
ldapsearch -LLL -x -H ldap://<DC-IP> -b "DC=empresa,DC=local" "(objectclass=*)"

Con usuario autenticado
ldapwhoami -x -H ldap://<DC-IP> -D "usuario@empresa.local" -w "password"

Todo el Dominio
ldapsearch -LLL -x -H ldap://<DC-IP> \
-D "usuario@empresa.local" -w "password" \
-b "DC=empresa,DC=local"

Enumerar todos los usuarios
ldapsearch -LLL -x -H ldap://<DC-IP> \
-D "usuario@empresa.local" -w "password" \
-b "DC=empresa,DC=local" \
"(&(objectClass=user)(!(objectClass=computer)))" cn

