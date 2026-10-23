# HYPR Windows Desktop App — Technical Report

**Compiled:** 2026-10-23
**Scope:** The HYPR Windows desktop client — known across its lifetime as the **Workforce Access Client (WFA)**, then **HYPR Passwordless for Windows**, under the **HYPR Authenticate** platform brand. Covers what it is, how it works, and how it integrates with ("hooks into") Windows.
**Audience:** Security research / technical.

> **Method & reliability note.** Assembled from public sources: HYPR's own documentation (`docs.hypr.com` = current, `classicdocs.hypr.com` = legacy v7–v8.x, `get.hypr.com` = end-user guides, release-notes tree), HYPR marketing (`hypr.com`), Microsoft/third-party references on Windows credential providers, and industry press. The environment's network policy blocked direct page fetches, so HYPR doc content here was read via search-engine extraction plus a delegated research pass over the same doc tree, **not** full-text retrieval of every page. **Vendor claims are labeled.** Inferences are flagged. HYPR versions behavior per release (7.x–11.x observed); value names, defaults, platform minimums, and UI changed across releases — confirm anything version-sensitive against the docs for the specific deployed build. The **current docs path segment for all Windows/macOS client pages is `pwc` ("Passwordless Client")**: `/docs/auth/authInstallCfg/authInstallCfgPwc/...`. Source URLs: [§17](#17-sources).

---

## 1. Disambiguation — which "hypr" this is

| Name | What it is | Relevance |
|---|---|---|
| **HYPR (HYPR Corp)** | Commercial enterprise *passwordless / identity assurance* vendor. Its Windows endpoint software is the subject here. | **This report.** |
| **Hyprland** | A Linux (Wayland) tiling window manager. No native Windows build. | Name collision only. |
| **Hyprspace** | A libp2p/IPFS peer-to-peer VPN project. Third-party "Windows download" listings (e.g. onworks.net) are unofficial. | Name collision only. |

Everything below concerns **HYPR Corp's Windows authentication client**.

---

## 2. Product naming history & lineage

The Windows client is **one continuous product line under three names**:

| Era | Name | Marker |
|---|---|---|
| ~2020–2023 (v7.x–8.x) | **HYPR Workforce Access Client (WFA)** | Legacy name throughout `classicdocs`. |
| Dec 2023+ (v8.7.0+) | **HYPR Passwordless for Windows** | Formal rename. The **8.7.0 release notes carry a naming-mapping table** mapping "HYPR Passwordless for Windows" (new) ← "HYPR Workforce Access Client for Windows" (legacy). Authoritative rename marker. |
| 2025+ (10.5.x+) | **"Authenticate for Windows"** component tag | **"HYPR Authenticate"** is the umbrella **platform/app brand**, not a binary rename; release-note tables still ship "HYPR Passwordless for Windows" through 10.5.x–11.3.x. |

**Key nuance:** the WFA → Passwordless rename is real and dated (8.7.0). The "Authenticate" shift is a platform/marketing layer. The **MSI is still literally `WorkforceAccess_x64.msi` in current 11.x install docs** — a vestige of the original WFA name. (Per delegated research over the current docs tree; version/date specifics below are HYPR-doc-sourced but were not re-fetched page-by-page here.)

**Platform family (admin-facing via Control Center unless noted):**
- **HYPR Control Center** — web admin hub on a HYPR tenant; manages app policies, users, devices, SSO/federation, mobile branding; hosts installer downloads (Workstation → Desktop Client).
- **HYPR Device Manager** — self-service web UI for users to pair/unpair trusted devices; underpins "Single Registration."
- **HYPR Mobile App (HYPR Authenticate app)** — phone as FIDO2 passkey; the authenticator for the desktop client.
- **HYPR Adapt** — risk / adaptive-auth engine; ingests mobile + workstation + behavior/browser/endpoint signals; can route risky sessions to Affirm and feed SIEM/SOAR.
- **HYPR Affirm** — workforce **identity verification** (onboarding, recovery, help-desk resets, step-up): gov-ID checks, biometric/passive liveness, directory validation.
- **HYPR Server / SDKs** — backend (Server 10.0.0 requires JDK 17+) and mobile/web SDKs.
- **"HYPR Authenticate"** = current passwordless-MFA platform brand; **"HYPR Identity Assurance"** = broader lifecycle offering.
([about](https://hypr.com/about), [press release](https://www.hypr.com/press-releases/hypr-advances-mission-to-bring-workforce-identification-to-scale), [intro-auth / four components](https://docs.hypr.com/docs/auth/intro-auth/))

**Company:** Founded **2014** (formerly "HYPR, the Passwordless Company"), Holmdel, NJ. Bojan Simic reported as CEO. Founder attribution (incl. George Avetisov) and cumulative funding figures vary across secondary databases — treat as **unverified** without a primary filing. ([CB Insights](https://www.cbinsights.com/company/hypr), [Help Net Security](https://www.helpnetsecurity.com/2022/12/02/hypr-funding/))

---

## 3. What the Windows desktop app *is*

A Windows endpoint agent that lets a user **sign in to / unlock a Windows workstation without typing an AD password**, authenticating instead with one of:

1. a paired **smartphone** (HYPR Mobile App) — FIDO2 passkey with face/fingerprint/PIN;
2. a **FIDO2 security key / PIV smart card**;
3. an **offline PIN** when offline.

Architecturally HYPR documents **"four main components"**: Control Center + Device Manager + Mobile App + the Passwordless desktop client. The desktop client is **certificate-based**: when a mobile device is paired to the workstation, **a virtual smartcard is created on the machine** and used to perform Windows logon/unlock. ([client overview](https://docs.hypr.com/hyprcloud/docs/workforce-access-client-overview), [intro-auth](https://docs.hypr.com/docs/auth/intro-auth/), [requirements](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-requirements/))

**Install types:** **Quick** (no AD; skips the Environment-Setting dialog) vs **Advanced** (domain-join + AD CS; opens the Environment-Setting dialog; unlocks Offline Mode etc.). ([installing with the UI](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-installation-ui/))

---

## 4. How it hooks into Windows — architecture & OS integration

HYPR integrates through **standard, Microsoft-sanctioned extension points**, not kernel hooks. The load-bearing pieces: a **Credential Provider**, a **Key Storage Provider**, a **Windows NT service**, and **certificate (smart-card-style) logon via AD CS** implemented as a **virtual smartcard**.

### 4.1 Credential Provider (CP) — the LogonUI hook
General Windows behavior HYPR conforms to:
- A credential provider is an in-process **COM** object implementing `ICredentialProvider` in a DLL, loaded by **`LogonUI.exe`** (and `consent.exe`) to render logon tiles on the Secure Desktop. It **collects** a credential and hands it to LSA — it does **not** authenticate itself. ([dennisbabkin CP internals](https://dennisbabkin.com/blog/?i=AAA12400), [MS EM blog](https://cloudblogs.microsoft.com/enterprisemobility/2009/07/10/testing-a-credential-provider/))
- **Registration (HKLM):** providers at `...\Microsoft\Windows\CurrentVersion\Authentication\Credential Providers\{CLSID}`; filters at `...\Authentication\Credential Provider Filters\{CLSID}`; DLL via the COM class `InprocServer32` (security best practice: absolute path, admin-write-only dir). `...\Authentication\LogonUI\LastLoggedOnProvider` holds the last provider GUID. A provider is disabled with `Disabled=1 (REG_DWORD)` under its CLSID (applies next session; Safe Mode loads only the built-in password provider). ([dennisbabkin](https://dennisbabkin.com/blog/?i=AAA12300), [JumpCloud](https://jumpcloud.com/support/find-the-default-windows-credential-provider))

HYPR-specific:
- HYPR ships its **own credential provider** (the HYPR logon tile: QR/push/security-key/offline-PIN). Its log file name is the registry value **`HyprCPLogFile`**.
- HYPR uses a **Credential Provider Filter** to suppress other tiles when enforcing passwordless; by default it filters the built-in **Password** and **Network Password** providers, and extra GUIDs can be added via **`HYPRPWDCREDPROVFILTER`** (registry *"Additional Password Provider GUIDs"*). HYPR warns misconfiguration **can remove all logon options** (self-lockout risk). **10.5.0** added *expanded* password-credential-provider filtering and an option to **always enforce passwordless (hide password tiles even with no HYPR device paired)**.
- **Conflicts:** HYPR requires no conflicting CP *filters* pre-exist, endpoint security must **allowlist HYPR**, and for smart-card enforcement the built-in Windows smart-card CP must remain available.
([troubleshooting](https://docs.hypr.com/hyprcloud/docs/workforce-access-client-troubleshooting), [passwordless enforcement](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/auth-install-cfg-pwc-adv-cfg-passwordless-enforcement/), [requirements](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-requirements/))

### 4.2 Key Storage Provider (KSP) — CNG hook
HYPR registers a **Key Storage Provider** in Windows **CNG** to surface HYPR-managed keys/certs (the virtual smartcard) to the crypto stack. Its log file name is registry value **`HyprKspLogFile`**. The certificate template specifies a **KSP** as the cryptographic provider under a **Smart Card Logon** profile. ([troubleshooting](https://docs.hypr.com/hyprcloud/docs/workforce-access-client-troubleshooting), [custom certificate template](https://docs.hypr.com/hyprcloud/v8.1.0/docs/workforce-access-client-custom-certificate-template))

### 4.3 Windows NT service
A **HYPR Windows NT service** runs the background agent (pairing state, push/QR handling, certificate lifecycle, offline-PIN cache, Control Center comms), with its own **log-level** registry setting distinct from CP/KSP. ([troubleshooting](https://docs.hypr.com/hyprcloud/docs/workforce-access-client-troubleshooting))

> **Gap (corroborated independently):** No public HYPR doc names the **Credential Provider DLL filename, the KSP binary, or the NT service's display/binary name / CLSIDs.** The mechanism is documented only *functionally* (virtual smartcard). Enumerate on a live install: `sc query` / `Get-CimInstance Win32_Service`; dump both `Authentication\...` registry keys; `certutil -csplist` for the CNG KSP.

### 4.4 Certificate logon via AD CS (the core auth mechanism — "virtual smartcard")
Instead of injecting a password, HYPR drives **certificate (smart-card) logon**, realized as a **virtual smartcard** created on the machine at pairing:
1. **Infra prereqs:** **Enterprise CA (AD CS, Windows Server 2008 R2+)** AD-integrated, issuing **user-auth certs**, with **autoenrollment**; workstation **domain-joined**. ([requirements](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-requirements/), [smart card playbook](https://docs.hypr.com/docs/11.1.0/playbooks/windows-passwordless-login-using-smartcard-playbook/))
2. **Custom template (default `hyprwin`, macOS `hyprmac`):** duplicated from **Smart Card User**; **Application Policies = `Client Authentication` + `Smart Card Logon` only**; **Subject = UPN**; provider = **KSP**; **RSA ≥2048 recommended** (1024 min historically) **or ECC**. If renamed, each workstation config (`certTemplate`) must match. ([custom certificate template](https://docs.hypr.com/hyprcloud/v8.1.0/docs/workforce-access-client-custom-certificate-template), [smart card playbook](https://docs.hypr.com/docs/11.1.0/playbooks/windows-passwordless-login-using-smartcard-playbook/))
3. **Domain controllers:** each needs a CA cert with **Smart Card Logon / KDC Auth** EKU or cert logon fails. ([playbook deployment](https://docs.hypr.com/docs/playbooks/playbooksDeployment/windows-passwordless-login-using-smartcard-playbook/))
4. **Trust & revocation:** issuing CA trusted by all domain systems; **CRL/OCSP reachable** by DCs and endpoints. ([smart card playbook](https://docs.hypr.com/docs/11.1.0/playbooks/windows-passwordless-login-using-smartcard-playbook/))
5. **Logon:** standard Windows smart-card-logon → **PKINIT** → Kerberos TGT, after the HYPR gesture (phone biometric/PIN, or security-key PIN) releases use of the private key. *(PKINIT sequence is standard Windows; HYPR docs cover the cert/EKU/DC prerequisites.)*
6. **Pairing token flow (vendor-stated):** pairing scans an on-screen **QR (expires ~1 min, ~3 min with "Require User Presence")**; an **install/pairing token** is verified against the HYPR Server and exchanged for a **device-unique security token** securing workstation↔mobile↔server traffic.

HYPR frames this as certificate-based auth that **removes the need for password resets**, recommending AD settings like *User cannot change password / Password never expires / Smart card required for interactive logon*. **No source indicates HYPR randomizes/scrambles the AD password** — the mechanism is certificate logon, not password rotation. ([passwordless enforcement](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/auth-install-cfg-pwc-adv-cfg-passwordless-enforcement/), [password reset config](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/auth-install-cfg-pwc-adv-cfg-password-reset-configuration-win/))

### 4.5 Registry footprint & logging
- **Hive:** `HKLM\SOFTWARE\HYPR Workforce Access`. As of **7.7.0**, relocated so **user-specific entries are Administrator-only** (hardening relevant to registry-ACL analysis).
- **Observed values:** `HyprCPLogFile`, `HyprKspLogFile`, NT-service log level; `HYPRPWDCREDPROVFILTER` ("Additional Password Provider GUIDs"); `securityKeyPinMinimumLength` (6/7/8, default 6); cert **revocation-check timeout** (~3000 ms) and **template-init timeout** (~30000 ms); offline-PIN config (count/validity/length); **"HYPR Protect Logs"** (log-folder ACL; **opt-in** — may not exist unless set at install).
- **Logs:** `C:\Program Files\HYPR\Log` (no `ProgramData` path confirmed). An **"In-App Logs Submission"** toggle (Control Center) chooses API-to-HYPR-Support vs local email client.
- **Diagnostics tool:** older builds launch it by **double-clicking the HYPR logo** in the client.
- **Direct registry edits of RP URL / App ID are discouraged** — changing them requires uninstall + reinstall.
([troubleshooting](https://docs.hypr.com/hyprcloud/docs/workforce-access-client-troubleshooting), [security best practices](https://docs.hypr.com/hyprcloud/docs/workforce-access-client-security-best-practices), [7.7.0 notes](https://docs.hypr.com/docs/releaseNotes/legacyVersions/770-release-notes/), [7.1.0 notes](https://docs.hypr.com:443/hyprcloud/docs/710-release-notes), [offline API](https://docs.hypr.com/docs/api/apiAuthentication/api-authentication-offline/))

---

## 5. Authentication methods (workstation logon)

| Method | Description | Notes |
|---|---|---|
| **Smartphone** | Phone = FIDO2 passkey; approve via face/fingerprint/PIN; login **initiated from phone** or by **QR scan**. | HYPR favors QR / mobile-initiated over plain push (push-fatigue resistance). ([app](https://www.hypr.com/platform/app), [tap-to-login API](https://docs.hypr.com/docs/api/apiAuthentication/api-authentication-tap-to-log-in/)) |
| **Security key / PIV smart card** | FIDO2 key or PIV smart card; client **stores a certificate on the device**; admin-governed per server/app. | Requires PIV; **Security Key Recovery Mode** fallback; client renews certs against AD CS after a valid PIN. ([security key support](https://docs.hypr.com/docs/guides/guidesFeatures/guidesFeaturesWS/guides-features-ws-security-key-support/), [cert renewal](https://docs.hypr.com/hyprcloud/v8.2.0/docs/workforce-access-client-advanced-config-certificate-renewal-security-keys-win)) |
| **Offline PIN** | Single-use **Decentralized PIN** generated in the mobile app for offline unlock. | Advanced install + prior pairing; see [§8](#8-offline-mode). ([offline API](https://docs.hypr.com/docs/api/apiAuthentication/api-authentication-offline/)) |

**Documented supported security keys (version-dependent):** YubiKey 5 Series (fw 5.x), **YubiKey Bio Multi-Protocol Edition**, IDEMIA ID-One on Cosmo 8.2, Feitian ePass K9 Plus / K40 Plus. **Breaking change at 10.5.0: HID C2300 cards dropped** (minidriver issue). YubiKey Bio covers Windows desktop login via **Smart Card/PIV + fingerprint** and web FIDO2. **Entra-joined devices:** "Start Pairing" can provision a FIDO2 key **directly into Entra ID**. ([security key support](https://docs.hypr.com/docs/guides/guidesFeatures/guidesFeaturesWS/guides-features-ws-security-key-support/), [YubiKey](https://hypr.com/integrations/yubikey))

---

## 6. Device pairing / registration
- **QR pairing:** scan an on-screen QR with the Mobile App; out-of-band channel; completes in seconds. ([register via web](https://get.hypr.com/register-smartphone-via-web), [register on Windows](https://get.hypr.com/register-smartphone-on-windows))
- **Magic Links:** if the app is installed, registration auto-starts without an external QR scan. ([Magic Links](https://docs.hypr.com/hyprcloud/docs/cc-adv-app-cfg-magic-links))
- **Single Registration (9.0.0+, beta):** a device paired to a web RP via Device Manager **auto-completes on the workstation** without a second pairing. Backed by the **Certificate Enrollment Service** ([§9.4](#94-certificate-enrollment-service-single-registration)). ([Device Manager](https://docs.hypr.com/docs/auth/authUserExp/authUserExpReg/auth-user-exp-reg-device-manager/))
- **De-pairing:** removing **all** pairings **re-enables password logon by default** (lockout/recovery consideration). ([passwordless enforcement](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/auth-install-cfg-pwc-adv-cfg-passwordless-enforcement/))

---

## 7. Protocol & cryptographic model
- **Standards:** FIDO2-certified; FIDO UAF referenced; web auth stated FIDO-Alliance-compliant for security keys. ([FIDO server](https://www.hypr.com/fido-server/), [security key support](https://docs.hypr.com/docs/guides/guidesFeatures/guidesFeaturesWS/guides-features-ws-security-key-support/))
- **Key model (vendor-stated):** public-key challenge-response; **private key in secure hardware on the device** (phone secure element/TEE, or key/PIV); "no shared secrets"; verification by signature vs registered public key. ([for security teams](https://www.hypr.com/for-security-teams), [FIDO server](https://www.hypr.com/fido-server/))
- **FIDO2 RP settings (Control Center):** discoverable-credential preference, user-verification mode, attestation type, **origin URL** for request validation. **11.3+**: legacy FIDO2 **MDS metadata-management APIs deprecated**, replaced by granular **AAGUID allow/deny** controls (Control-Center-side). ([FIDO2 settings](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgAppMgmt/auth-install-cfg-app-mgmt-fido2-settings/))
- **Federation / tokens:** **OIDC provider** (returns `id_token`), plus **SAML / OAuth2**. ([OIDC](https://docs.hypr.com/docs/11.1.0/int/int-openid-connect/), [security FAQ](https://docs.hypr.com/docs/guides/guidesFaq/guides-faq-security-infrastructure/))
- **Transport pinning:** SDKs/clients support **SSL/TLS public-key pinning** (`pinningHash`; comma-separated list, any match passes; `DISABLE` turns it off — the **disable-pinning option was added 10.5.0**). ([iOS SDK config](https://docs.hypr.com/docs/sdk/sdkHypr/sdkHyprModels/sdkHyprModelsIos/sdk-hypr-configuration-hypruseragent-data-model/))

> *"Decentralized / no shared secret" is a vendor framing consistent with FIDO/UAF device-bound keys; the on-device credential storage/crypto internals are not fully public.*

---

## 8. Offline mode
- **Purpose:** unlock/logon with no connectivity via a **single-use Decentralized PIN** generated in the mobile app. Advanced install only. ([requirements](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-requirements/))
- **Prereqs:** client already **paired**; offline auth **off by default**, enabled per **RP Application** in Control Center; **no conflicting credential-provider filters**.
- **Config sourcing:** offline-PIN config **read from the workstation registry** (cached count / validity days / PIN length); endpoint config first pulled from Control Center on the first successful online auth, then cached. ([offline API](https://docs.hypr.com/docs/api/apiAuthentication/api-authentication-offline/))
- **Documented ranges (Control Center UI):** PIN length **6–13**, number of offline PINs **10–25**, offline window **7–90 days**. ([workstation settings](https://classicdocs.hypr.com/hyprcloud/docs/cc-std-ws-settings), [using an offline PIN](https://get.hypr.com/using-an-offline-pin-win))

> *Research angle:* the offline path substitutes server verification/revocation with a bounded local PIN cache (registry-held, ACL-hardened at 7.7.0). The count/validity/length bounds define the offline brute-force/replay envelope.

---

## 9. Installation & deployment

### 9.1 Packaging & sourcing
- Windows: **MSI `WorkforceAccess_x64.msi`** (x64; name unchanged post-rebrand). **32-bit** existed through 8.3.x, **dropped at 8.4.0**. **ARM64** = **beta/private-preview, request-only from HYPR Support** (no published ARM64 MSI filename). ([installing manually](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-installation-manual))
- Downloaded from **Control Center → Workstation → Desktop Client** (Windows/Mac + Quick/Advanced). The download bundles MSI + config; **extract the zip first** — running the MSI from inside the archive means it "won't be able to read the `hypr.json` configuration file." ([installing with the UI](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-installation-ui/))

### 9.2 Config file — `hypr.json`
Plain JSON name/value pairs, **in the same folder as the MSI**; all optional unless marked required.

| MSI property | `hypr.json` key | Registry name | Meaning |
|---|---|---|---|
| `HYPRRP` | `rpUrl` *(required)* | "Relying Party Url" | HYPR instance URL; **must end in `/rp`** |
| `HYPRAPPID` | `appId` *(required)* | — | Control Center workstation app; standard `HYPRDefaultWorkstationApplication` |
| `HYPRHASH` | `pinningHash` | — | SSL/TLS public-key pin; list allowed; `DISABLE` off |
| `HYPRTEMPLATE` | `certTemplate` | — | AD template; default `hyprwin` (Win) / `hyprmac` (Mac) |
| `HYPRINSTALLTOKEN` | `installToken` | — | install/pairing token |
| `HYPRSUPPORT` | `supportEmail` | — | support contact |
| `HYPRPROXYSERVER`/`HYPRPROXYBYPASS` / `HYPRPROXYAUTOCONFIGURL` | — | — | proxy config / PAC URL |
| `HYPRNONPERSISTENTVDI=1` | — | reg value | non-persistent VDI deployment |
| — | `fullUI` | — | 1 shows / 0 hides Environment-Setting dialog (n/a with msiexec params) |
| — | `disablePasswordLogin` | (bit-flag) | set = passwordless mandatory; **cleared only via registry/GPO** |
| `HYPRCUSTOMLOGO`/`HYPRCUSTOMBACKGROUND` | `customLogo`/`customBackground` | — | branding |

([installing manually](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-installation-manual))

### 9.3 Install modes, silent install, upgrade, uninstall
- **Quick Install** — no params; skips Environment-Setting dialog; **workstation must NOT be AD-joined.**
- **Advanced Install** — opens **Environment Setting dialog** (RP URL, App ID, Public Key Pinning, Cert Template, Support Email, Install Token, Add to Start Menu, Non-persistent VDI, Disable Password Login); **requires domain-join + AD CS.** Fields normally pre-filled from `hypr.json`. **Restart recommended after install.**
- **Silent:** `msiexec.exe /qn /i WorkforceAccess_x64.msi` (reads adjacent `hypr.json`); or inline, e.g. `... HYPRAPPID="HYPRDefaultWorkstationApplication" HYPRRP="https://<tenant>.hypr.com/rp"`. **Elevated shell required** for all silent operations.
- **Upgrade:** `msiexec.exe /q /i WorkforceAccess.msi` **preserves device registration**; silent installs **reboot by default** → add **`/norestart`**. **From 10.5.1:** opt-in flag applies new installer/config values on upgrade; **RP URL + App ID always preserved** (RP URL, App ID, Custom Background/Logo ignored on upgrade).
- **Uninstall:** `msiexec.exe /x WorkforceAccess_x64.msi`, or installer UI, or **Control Panel → Apps & Features** (admin). **Removes all registry keys, install artifacts, and HYPR user accounts** on the workstation; **restart required**.
([installing manually](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-installation-manual), [removal](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-removal/), [upgrading on Windows](https://docs.hypr.com/hyprcloud/docs/upgrading-the-application-win))

### 9.4 Certificate Enrollment Service (Single Registration)
Separate component for **Web→Workstation Single Registration**:
- Package **`HyprEnrollmentService_x64.msi`** (silent via msiexec). Requires **.NET Framework 4.6.2**; host **domain-joined**; install on a **separate server (not the DC or the AD CS server)** in the same security tier.
- Params: Control Center URL, a Control-Center **admin RP-App access token**, cert template, SSL pin. RP-App feature flags: `WINDOWS_WEB_ENROLLMENT`, `ENROLLMENT_SERVICE`, `ENROLLMENT_SERVICE_APP` (11.x). Enrollment agent recommended to poll ~every 15 s.
([single-reg enrollment service](https://docs.hypr.com/docs/playbooks/playbookSingleReg/single-reg-enrollment-service/))

### 9.5 Enterprise rollout & VDI
- **Intune (documented, 10.1–11.x):** package as a **Win32 app** (MSI + `hypr.json` + optional branding zip), convert with the **Win32 Content Prep Tool** → `.intunewin`, upload. **SCCM/MECM & GPO** listed as supported but **no MECM task-sequence guidance published** (gap). ([Intune deployment](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/auth-install-cfg-pwc-adv-cfg-deployment-via-intune/))
- **VDI golden image (guide = 11.3.0):** remove any prior client → install with **`HYPRNONPERSISTENTVDI=1`** (UI checkbox / msiexec param / registry value) → reboot → **Sysprep /generalize /shutdown** → capture. Each clone needs a **different Machine SID or Computer Name** than the master. ([golden image VDI](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/auth-install-cfg-pwc-adv-cfg-creating-golden-image-vdi-win/))

---

## 10. Features (workstation)
- **HYPRspeed (Desktop SSO)** — auto SSO after mobile-app unlock; **session ends on workstation lock as of 11.1.1**.
- **Remote Lock / Unlock** — lock/unlock a domain-joined Windows machine from the Mobile App. ([remote lock](https://www.hypr.com/remote-lock/), [ws lock/unlock](https://classicdocs.hypr.com/hyprcloud/v8.1.0/docs/features-ws-lock-unlock))
- **RDP (passwordless)** — domain-joined users authenticate via Mobile App over the Microsoft RDP client; **HYPR client not required on the remote target**; remote account may differ but needs target permissions. **8.3.0-era constraint: RDP supported only with NLA disabled** — security-relevant (removes CredSSP pre-auth); confirm against current version. ([accessing a remote desktop](https://classicdocs.hypr.com/hyprcloud/docs/workforce-access-client-usage-accessing-remote-desktop), [8.3.0 notes](https://docs.hypr.com/docs/releaseNotes/legacyVersions/830-release-notes/))
- **Roaming Users (Windows)** — pair once, then log into any domain-joined desktop by scanning a login-screen QR; **off by default**. ([accessing a remote desktop](https://classicdocs.hypr.com/hyprcloud/v8.1.0/docs/workforce-access-client-usage-accessing-remote-desktop))
- **Offline Mode / Single Registration** — see [§8](#8-offline-mode), [§6](#6-device-pairing--registration).
- **Passwordless enforcement** — hide password tiles via default + extra CP filtering, per-provider `Disabled=1`, or GPO **Exclude credential providers** (CLSID list). ([passwordless enforcement](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/auth-install-cfg-pwc-adv-cfg-passwordless-enforcement/))

---

## 11. Version timeline (release notes 7.x → 11.x)

Release-notes tree: `docs.hypr.com/docs/releaseNotes/` (`legacyVersions/` + `currentlySupported/`); pages `NNN-release-notes`. **Channels:** **Standard** = monthly; **Enterprise** = quarterly (stable); **Internal** = HYPR's own environments. (Dates/specifics below are from HYPR release-note pages via delegated research; treat exact dates as HYPR-sourced, not independently re-fetched here.)

| Version (date) | Windows-relevant change |
|---|---|
| **7.1.0** (Mar 2022) | **Windows 7 support dropped** (client now Win 10/11). |
| **7.7.0** (Sep 2022) | **Registry hardening** — HYPR key relocated so user-specific entries are **Administrator-only**; improved TPM handling. |
| **7.9.0** (Nov 2022) | Pairing, log-submission, QR-timeout, AD CS fixes. |
| **8.0.0** (Feb 2023) | TPM attestation verifier matches **EC** public keys. |
| **8.4.0** (Aug 2023) | **32-bit Windows discontinued** (64-bit only). |
| **8.7.0** (Dec 2023) | **RENAME → "HYPR Passwordless for Windows"** (naming-mapping table). |
| **9.0.0** (Feb 2024) | **Single Registration: Web→Workstation.** |
| **9.7.0** (Dec 2024) | Dedicated **"Breaking Changes"** section introduced. |
| **10.0.0** (Feb 2025) | Security-key support (YubiKey 5 Plus, Feitian K9 Plus / K40 Plus). |
| **10.5.0** (Sep 2025) | **Azure Trusted Signing** code signing; **option to disable SSL pinning**; **expanded password-CP filtering**; **always-enforce passwordless**; expanded security keys; **BREAKING: HID C2300 dropped**. |
| **10.5.1** | **Opt-in: apply new installer/config values on upgrade** (RP URL + App ID always preserved); cert re-enrollment for Web Registration. |
| **11.1.0** (Mar 2026) | **BREAKING: min Windows → 10 "2004"; 1803/1809 dropped** (22H2 recommended). |
| **11.1.1 / 11.1.2** | Lock ends HYPRspeed SSO session; security hardening. |
| **11.3.x** (current GA) | min Win 10 "2004"/11; **FIDO2 MDS metadata APIs deprecated** → AAGUID allow/deny. |

Every Windows upgrade entry notes *"Reboot required if upgrading from 7.6 or below."*

---

## 12. System requirements (current)
- **Windows 11** (incl. 25H2); **Windows 10 Pro 2004+** (minimum rose over time, reaching **2004 at 11.1.0**; earlier docs listed 1607+/1803+). **ARM64 = beta on request.**
- **macOS** client: macOS 26 / 15 / 14.1 / 13 / 12 / 11 (14.x needs HYPR 8.5.1+); template default `hyprmac`.
- **Advanced install prereqs:** domain-joined + **AD CS (Win Server 2008 R2+)**, Enterprise CA AD-integrated, cert autoenrollment, reachable **CRL/OCSP**, dedicated template with **Client Authentication + Smart Card Logon** EKUs, **KSP** provider, **RSA 2048 or ECC**.
([supported platforms](https://docs.hypr.com/docs/guides/guidesHyprSupported/guides-hypr-supported-platforms/), [requirements](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-requirements/))

---

## 13. Backend / control plane
- **Control Center** defines RP Applications, workstation settings (offline, roaming, lock/unlock, enforcement, log submission), FIDO2 policy; distributes installers. ([Control Center](https://docs.hypr.com/hyprcloud/v8.1.0/docs/cc-std))
- **RP + FIDO server** validate assertions; RP config includes the **origin URL**. ([FIDO server](https://www.hypr.com/fido-server/), [FIDO2 settings](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgAppMgmt/auth-install-cfg-app-mgmt-fido2-settings/))
- **Topologies:** HYPR Cloud (`hyprcloud`) and Control-Center/on-prem; backend **Server 10.0.0 requires JDK 17+**.
- **Third-party orchestration:** Strata Maverics connector (`hyprDomain`, `hyprAppID`, `accessToken`); referenced by BeyondTrust PRA. ([Strata](https://docs.strata.io/docs/hypr), [BeyondTrust PRA](https://docs.beyondtrust.com/pra/v25.3/docs/hypr))

---

## 14. Security-research observations (neutral, mechanism-focused)
1. **CP/KSP load in SYSTEM/pre-logon context** (`LogonUI.exe`/LSA, Secure Desktop). The DLL `InprocServer32` path + on-disk ACLs are the primary tamper surface; a writable provider DLL path is a classic SYSTEM escalation vector. ([dennisbabkin](https://dennisbabkin.com/blog/?i=AAA12400))
2. **CP filter misconfiguration = DoS/lockout.** HYPR itself warns filtering wrong GUIDs removes all tiles; keep a tested recovery path (Safe Mode = built-in password provider; de-pair re-enables password login). ([passwordless enforcement](https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/auth-install-cfg-pwc-adv-cfg-passwordless-enforcement/))
3. **PKI is the root of trust.** Security reduces to the AD CS template scope (EKUs, UPN subject, ≥2048-bit), DC cert hygiene, and CRL/OCSP reachability. A permissive/mis-scoped template or enrollment-agent weakness is the high-value target — standard AD CS (ESC-class) analysis applies. The **Certificate Enrollment Service holds a Control-Center admin access token** and sits on its own domain-joined server — another credential/host worth scoping. ([custom certificate template](https://docs.hypr.com/hyprcloud/v8.1.0/docs/workforce-access-client-custom-certificate-template), [single-reg enrollment service](https://docs.hypr.com/docs/playbooks/playbookSingleReg/single-reg-enrollment-service/))
4. **RDP + NLA.** The documented (8.3.0-era) requirement to disable NLA removes CredSSP pre-auth on the listener — weigh vs the passwordless benefit; confirm current behavior. ([8.3.0 notes](https://docs.hypr.com/docs/releaseNotes/legacyVersions/830-release-notes/))
5. **Offline PIN bounds** (len 6–13, 10–25 PINs, 7–90 days) define the offline guessing/replay envelope; the registry hive's ACLs (7.7.0) gate local access. ([offline API](https://docs.hypr.com/docs/api/apiAuthentication/api-authentication-offline/))
6. **Log confidentiality** — "Protect Logs" ACL is **opt-in**; check a default deployment. ([security best practices](https://docs.hypr.com/hyprcloud/docs/workforce-access-client-security-best-practices))
7. **Pinning can be disabled (10.5.0+)** — a deployment with `pinningHash=DISABLE` weakens MITM resistance to the Control Center/RP. ([iOS SDK config](https://docs.hypr.com/docs/sdk/sdkHypr/sdkHyprModels/sdkHyprModelsIos/sdk-hypr-configuration-hypruseragent-data-model/))
8. **Push vs QR** — HYPR deprecates plain push in favor of QR/mobile-initiated with local user verification; relevant to phishing/social-engineering evaluation. ([tap-to-login API](https://docs.hypr.com/docs/api/apiAuthentication/api-authentication-tap-to-log-in/))

---

## 15. Vulnerability & advisory history (independent / third-party)

This is the material that does **not** come from HYPR's own product docs — CVE records, a third-party pentest advisory, and an operational incident. It is the most useful independent signal for a researcher, and also the most **scrubbed by aggregators**: CVSS scores and affected-version ranges below are drawn from CVE aggregators (OpenCVE, Feedly, INCIBE, CyberStrike, GCVE, imfht) because the environment blocked direct fetches of **NVD** and HYPR's own **[security advisories page](https://hypr.com/security-advisories)**. **Treat all scores/ranges as leads to confirm against NVD and HYPR's advisory page** — aggregators disagree sharply (one CVE below is rated 3.7 *Low* by one source and 8.8 *High* by another). HYPR acts as its own **CNA**. Aggregators list on the order of **~18 HYPR CVEs** total. ([OpenCVE — HYPR](https://app.opencve.io/cve/?vendor=hypr), [OpenCVE — Workforce Access](https://app.opencve.io/cve/?product=workforce_access&vendor=hypr), [Feedly — HYPR](https://feedly.com/cve/vendors/hypr))

### 15.1 CVE table (Windows-relevant first)

| CVE | Product / platform | Class (CWE) | Affected → fixed | Aggregator CVSS (conflicting) | Notes |
|---|---|---|---|---|---|
| **CVE-2022-3258** | Workforce Access **(Windows)** | Incorrect permission assignment → **authentication abuse** | `< 7.7.1` | **3.7 Low ↔ 8.8 High** (wide disagreement) | CISA SSVC: exploitation *none*, *not automatable*. ([NVD](https://nvd.nist.gov/vuln/detail/CVE-2022-3258), [INCIBE](https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2022-3258)) |
| **CVE-2023-5097** | Workforce Access **(Windows)** | **Path traversal** / improper input validation (CWE-22) | `< 8.7` | 5.5 ↔ 7.0 | ([NVD](https://nvd.nist.gov/view/vuln/detail?vulnId=CVE-2023-5097)) |
| **CVE-2023-6334** | Workforce Access **(Windows)** | **Buffer overflow** | `< 8.7` | 5.3 ↔ 7.8 | Local/adjacent + some privileges. ([CyberStrike](https://cyberstrike.io/cve/CVE-2023-6334/)) |
| **CVE-2023-6335** | Workforce Access **(Windows)** | **Link following** (CWE-59) | `< 8.7` | 6.4 ↔ 7.8 | Pub. 2024-01-16; no known exploit. ([INCIBE](https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2023-6335)) |
| **CVE-2023-6336** | Workforce Access **(macOS)** | **Link following** (CWE-59) | `< 8.7` | 7.2 (HYPR CNA) | Windows twin = CVE-2023-6335. ([NVD](https://nvd.nist.gov/vuln/detail/cve-2023-6336)) |
| **CVE-2024-0068** | Workforce Access **(macOS)** | **Link following** (CWE-59) | `< 8.7.1` | 5.5 | ([OpenCVE](https://app.opencve.io/cve/?vendor=hypr)) |
| **CVE-2023-0834** | Workforce Access **(macOS)** | Incorrect permission assignment → **privilege escalation** | `6.12 → < 8.1` | 7.0 ↔ 9.8 | ([GCVE](https://db.gcve.eu/cve/CVE-2023-0834)) |
| **CVE-2022-2192** | **HYPR Server** | **Forced browsing** (CWE-425) → **privilege escalation** | `6.10 → 6.15.1` | 7.5 ↔ 8.8 | Attacker with a valid one-time **recovery token** tampers with the path on the **Magic Link** page to elevate. ([INCIBE](https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2022-2192), [GCVE](https://db.gcve.eu/vuln/cve-2022-2192)) |
| **CVE-2026-4522** *(aggregator id)* | HYPR Passwordless **(Windows)** | **Missing authentication → credential interception** | `< 11.1.1` *(aggregator)* vs **fixed 11.3.0** *(vendor changelog, 2026-06-24)* | — | **Version conflict unresolved** — confirm against HYPR's advisory. ([changelog](https://docs.hypr.com/docs/releaseNotes/release-notes-changelog/)) |

Security-relevant pattern: the pre-8.7 Workforce Access cluster (**path traversal, buffer overflow, two link-following, incorrect-permission**) are **local privilege / file-handling** issues in the Windows/macOS client — consistent with a privileged installer + a service/credential-provider running in SYSTEM context, where symlink/permission/path handling is the classic LPE surface. Most were remediated in the **8.7** generation (the same release that renamed the client).

### 15.2 Third-party pentest advisory — Compass Security CSNC-2022-003
- **"HYPR Workforce Access / Account Takeover,"** credited to **Philipp Mao**, dated **2022-10-13**, listed on Compass Security's public advisories index; a related 2022 advisory text references **account takeover in HYPR Workforce Access `< 6.18.0`** with "update to newest version" as the fix. ([Compass Security advisories](https://www.compass-security.com/en/research/advisories))
- **Root cause not retrievable** here (the advisory `.txt`/PDF was network-blocked). It plausibly corresponds to the 2022 **Magic Link / recovery-token** privilege-escalation class (cf. CVE-2022-2192), but that linkage is **unconfirmed** — the advisory itself should be read directly. This is the one substantive piece of **independent** (non-vendor) HYPR security research surfaced.

### 15.3 Operational incident
- HYPR reported an incident **2026-02-23 → resolved 2026-03-13**: **Workforce Access Clients were unavailable for download via Control Center**; admins were directed to HYPR Support for alternative download methods. (Availability, not a vulnerability.) ([status/incident tracker](https://pingoru.io/providers/hypr/incidents/107902))

### 15.4 What public blogs did *not* yield
Searches for an independent **reverse-engineering / DLL-and-service-name writeup**, a **HYPR engineering blog / Medium deep dive**, and **Reddit/Spiceworks deployment threads** came back **empty** — no non-vendor source names the credential-provider DLL, KSP binary, or NT service. General Windows-credential-provider tradecraft exists (e.g. `logonui.exe` COM-load + `CredProvDataModel.dll` hooking PoCs, MITRE **T1556**/**T1555**), but none is HYPR-specific. This *strengthens* the §16 conclusion that the client's binary/service internals are simply **not public** and require a lab teardown. ([LogonStealer PoC](https://github.com/S12cybersecurity/LogonStealer), [dennisbabkin: sequence of calls](https://dennisbabkin.com/blog/?t=sequence-of-calls-to-credential-provider-in-windows), [MITRE T1556.008](https://attack.mitre.org/techniques/T1556/008/))

---

## 16. Gaps / not independently verified
- **CP DLL filename, KSP binary, NT service name/CLSIDs** — not public (both this pass and a delegated pass confirm only a *functional* "virtual smartcard" description). Enumerate on a live install (`sc query`, registry dump, `certutil -csplist`).
- **No MECM/SCCM task-sequence guidance** (only Intune documented).
- **10.5.1 opt-in upgrade flag's exact name/registry path/value** not in summary notes.
- **ARM64 MSI filename / Apple-Silicon distribution** not documented.
- **Current NLA-disable RDP constraint** validity in the latest release.
- **On-device credential storage/crypto internals** (FIDO/UAF framing; "decentralized/no shared secret" = vendor claim).
- **Founder/funding** specifics conflict across secondary sources; minor date conflict on 8.7.0 (Dec 6 vs Dec 13).
- **All HYPR doc content here was read via search extraction / delegated research, not direct full-page fetch** (environment network policy blocked `docs.hypr.com` et al.). To raise to primary-source fidelity, broaden the environment's Network access (title-bar cloud-environment menu → Edit → add `docs.hypr.com`, `classicdocs.hypr.com`, `v9docs.hypr.com`, `www.hypr.com`, `get.hypr.com`, `learn.microsoft.com` under Allowed domains; https://code.claude.com/docs/en/cloud-environments#network-access), then re-fetch the §17 URLs in full and spot-check exact dates, value names, the release-note tables, and the §15 CVE scores/ranges against NVD.
- **CVE scores and affected-version ranges in §15 are aggregator-sourced and conflicting** — NVD and HYPR's advisory page were network-blocked and must be used as the authority. The **Compass Security CSNC-2022-003** root cause and the **CVE-2026-4522 vs 11.3.0** fixed-version conflict are unresolved.

---

## 17. Sources

**HYPR — official docs (current `pwc` tree):**
- Windows Installation & Configuration — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcInstallation/auth-install-cfg-pwc-installation-windows/
- Installing Manually (`hypr.json`, MSI props) — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-installation-manual
- Installing with the UI (Quick/Advanced) — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-installation-ui/
- Removal — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-removal/
- Requirements — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-requirements/
- Desktop Client — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-desktop-client/
- Upgrade — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/auth-install-cfg-pwc-upgrade/
- Passwordless Enforcement — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/auth-install-cfg-pwc-adv-cfg-passwordless-enforcement/
- Password Reset Config (Win) — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/auth-install-cfg-pwc-adv-cfg-password-reset-configuration-win/
- Custom Certificate Template (current) — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/authInstallCfgPwcAdvCfgCerts/auth-install-cfg-pwc-adv-cfg-certs-custom-template/
- Intune Deployment — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/auth-install-cfg-pwc-adv-cfg-deployment-via-intune/
- Golden Image VDI — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgPwc/authInstallCfgPwcAdvCfg/auth-install-cfg-pwc-adv-cfg-creating-golden-image-vdi-win/
- Security Key Support — https://docs.hypr.com/docs/guides/guidesFeatures/guidesFeaturesWS/guides-features-ws-security-key-support/
- Supported Platforms — https://docs.hypr.com/docs/guides/guidesHyprSupported/guides-hypr-supported-platforms/
- intro-auth (four components) — https://docs.hypr.com/docs/auth/intro-auth/
- Offline API Authentication — https://docs.hypr.com/docs/api/apiAuthentication/api-authentication-offline/
- Tap-to-Login API — https://docs.hypr.com/docs/api/apiAuthentication/api-authentication-tap-to-log-in/
- FIDO2 Settings — https://docs.hypr.com/docs/auth/authInstallCfg/authInstallCfgAppMgmt/auth-install-cfg-app-mgmt-fido2-settings/
- OIDC integration (11.1.0) — https://docs.hypr.com/docs/11.1.0/int/int-openid-connect/
- Security Infrastructure FAQ — https://docs.hypr.com/docs/guides/guidesFaq/guides-faq-security-infrastructure/
- iOS SDK config (pinning) — https://docs.hypr.com/docs/sdk/sdkHypr/sdkHyprModels/sdkHyprModelsIos/sdk-hypr-configuration-hypruseragent-data-model/
- Device Manager — https://docs.hypr.com/docs/auth/authUserExp/authUserExpReg/auth-user-exp-reg-device-manager/
- Smart Card Login Playbook (11.1.0) — https://docs.hypr.com/docs/11.1.0/playbooks/windows-passwordless-login-using-smartcard-playbook/
- Smart Card Playbook (deployment) — https://docs.hypr.com/docs/playbooks/playbooksDeployment/windows-passwordless-login-using-smartcard-playbook/
- Single Registration Enrollment Service — https://docs.hypr.com/docs/playbooks/playbookSingleReg/single-reg-enrollment-service/
- Release Notes index — https://docs.hypr.com/docs/releaseNotes/ (key: 710 Win7-drop, 770 registry-relocate, 840 32-bit-drop, 870 rename, 900 single-reg, 1050, 1051, 1110, 1133)

**HYPR — hyprcloud / classic / versioned:**
- Workforce Access requirements — https://docs.hypr.com/hyprcloud/docs/workforce-access-requirements
- Client overview — https://docs.hypr.com/hyprcloud/docs/workforce-access-client-overview
- Workstation Troubleshooting (logs/registry) — https://docs.hypr.com/hyprcloud/docs/workforce-access-client-troubleshooting
- Security Best Practices (Protect Logs) — https://docs.hypr.com/hyprcloud/docs/workforce-access-client-security-best-practices
- Custom Certificate Template (8.1.0) — https://docs.hypr.com/hyprcloud/v8.1.0/docs/workforce-access-client-custom-certificate-template
- Certificate Renewal for Security Keys (8.2.0) — https://docs.hypr.com/hyprcloud/v8.2.0/docs/workforce-access-client-advanced-config-certificate-renewal-security-keys-win
- Accessing a Remote Desktop — https://classicdocs.hypr.com/hyprcloud/docs/workforce-access-client-usage-accessing-remote-desktop
- Workstation Lock/Unlock — https://classicdocs.hypr.com/hyprcloud/v8.1.0/docs/features-ws-lock-unlock
- Workstation Settings (offline params) — https://classicdocs.hypr.com/hyprcloud/docs/cc-std-ws-settings
- Control Center: Standard Mode — https://docs.hypr.com/hyprcloud/v8.1.0/docs/cc-std
- Magic Links — https://docs.hypr.com/hyprcloud/docs/cc-adv-app-cfg-magic-links
- Upgrading on Windows — https://docs.hypr.com/hyprcloud/docs/upgrading-the-application-win
- Removing the application (legacy) — https://classicdocs.hypr.com/hyprcloud/docs/removing-the-application-win
- 8.3.0 release notes (RDP/NLA) — https://docs.hypr.com/docs/releaseNotes/legacyVersions/830-release-notes/
- 7.7.0 release notes (registry) — https://docs.hypr.com/docs/releaseNotes/legacyVersions/770-release-notes/
- 7.1.0 release notes (Win7 drop, diagnostics) — https://docs.hypr.com:443/hyprcloud/docs/710-release-notes

**HYPR — marketing / user guide:**
- About / Identity Assurance — https://hypr.com/about
- For Security Teams — https://www.hypr.com/for-security-teams
- Authenticate App — https://www.hypr.com/platform/app
- Passwordless Remote Login — https://www.hypr.com/platform/technology/remote-login
- Remote Lock — https://www.hypr.com/remote-lock/
- FIDO Server — https://www.hypr.com/fido-server/
- Passwordless Offline Mode — https://www.hypr.com/passwordless-offline-mode
- YubiKey integration — https://hypr.com/integrations/yubikey
- "Advances Mission…" press release — https://www.hypr.com/press-releases/hypr-advances-mission-to-bring-workforce-identification-to-scale
- Register smartphone via web — https://get.hypr.com/register-smartphone-via-web
- Register smartphone on Windows — https://get.hypr.com/register-smartphone-on-windows
- How to pair your smartphone — https://get.hypr.com/how-to-pair-your-smartphone
- Using an offline PIN (Win) — https://get.hypr.com/using-an-offline-pin-win

**Windows credential-provider / platform references:**
- dennisbabkin.com — CP internals/registration — https://dennisbabkin.com/blog/?i=AAA12400 · https://dennisbabkin.com/blog/?i=AAA12300
- Microsoft EM blog — Testing a credential provider — https://cloudblogs.microsoft.com/enterprisemobility/2009/07/10/testing-a-credential-provider/
- JumpCloud — Default Windows credential provider — https://jumpcloud.com/support/find-the-default-windows-credential-provider

**Third-party / company data (secondary):**
- CB Insights — HYPR — https://www.cbinsights.com/company/hypr
- Help Net Security — HYPR funding — https://www.helpnetsecurity.com/2022/12/02/hypr-funding/
- Strata Maverics HYPR connector — https://docs.strata.io/docs/hypr
- BeyondTrust PRA — HYPR — https://docs.beyondtrust.com/pra/v25.3/docs/hypr

**Vulnerability / advisory sources (§15):**
- HYPR Security Advisories (vendor CNA page) — https://hypr.com/security-advisories
- HYPR release-notes changelog — https://docs.hypr.com/docs/releaseNotes/release-notes-changelog/
- OpenCVE — HYPR — https://app.opencve.io/cve/?vendor=hypr
- OpenCVE — Workforce Access — https://app.opencve.io/cve/?product=workforce_access&vendor=hypr
- Feedly CVE — HYPR — https://feedly.com/cve/vendors/hypr
- imfht CVE tracker — HYPR (18 CVEs) — https://cve.imfht.com/vendor/HYPR?lang=en
- Exploit-Intel — HYPR — https://exploit-intel.com/vendor/HYPR
- NVD — CVE-2022-3258 — https://nvd.nist.gov/vuln/detail/CVE-2022-3258
- NVD — CVE-2023-5097 — https://nvd.nist.gov/view/vuln/detail?vulnId=CVE-2023-5097
- NVD — CVE-2023-6336 — https://nvd.nist.gov/vuln/detail/cve-2023-6336
- INCIBE — CVE-2023-6335 — https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2023-6335
- INCIBE — CVE-2022-2192 — https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2022-2192
- INCIBE — CVE-2022-3258 — https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2022-3258
- GCVE — CVE-2023-0834 — https://db.gcve.eu/cve/CVE-2023-0834
- GCVE — CVE-2022-2192 — https://db.gcve.eu/vuln/cve-2022-2192
- CyberStrike — CVE-2023-6334 — https://cyberstrike.io/cve/CVE-2023-6334/
- Compass Security — public advisories index (CSNC-2022-003, "HYPR Workforce Access / Account Takeover") — https://www.compass-security.com/en/research/advisories
- HYPR incident (client download outage, Feb–Mar 2026) — https://pingoru.io/providers/hypr/incidents/107902

**Windows credential-provider tradecraft (general, not HYPR-specific):**
- S12 — LogonStealer (CP-abuse PoC hooking `CredProvDataModel.dll`) — https://github.com/S12cybersecurity/LogonStealer
- dennisbabkin — Sequence of calls to a credential provider — https://dennisbabkin.com/blog/?t=sequence-of-calls-to-credential-provider-in-windows
- MITRE ATT&CK T1556.008 (Modify Authentication Process: Network Provider DLL) — https://attack.mitre.org/techniques/T1556/008/

---
*Prepared from public sources; vendor claims labeled inline. Version-sensitive details (dates, value names, platform minimums, release-note tables) should be re-confirmed against the documentation for the specific HYPR client build in use.*
