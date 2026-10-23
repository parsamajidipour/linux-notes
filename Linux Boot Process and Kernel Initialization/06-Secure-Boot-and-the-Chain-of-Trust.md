# Secure Boot and the Chain of Trust

Secure Boot answers a narrow but important question: should this executable image be allowed to run in the pre-operating-system chain?

It does not encrypt the disk. It does not prove that every configuration file is trustworthy. It does not guarantee that authorized code contains no vulnerability. It does not automatically attest the machine to a remote service. Those are different security properties.

A useful Secure Boot design begins with a threat model and follows every executable and mutable input from firmware to the final kernel policy. This chapter builds that chain, explains the UEFI key databases and common Linux shim path, and separates verified boot from measured boot, kernel lockdown, module signing, and storage encryption.

---

## 1. Start with the attacker

Imagine an attacker who can modify the unencrypted EFI System Partition but cannot extract a full-disk encryption key. Without code verification, the attacker can replace the loader with a program that imitates the unlock prompt, records the passphrase, and then starts the real system. Disk encryption protected offline data but did not authenticate the program that requested the key.

Secure Boot can prevent that replacement if firmware refuses the modified first image and every later stage refuses unauthorized successors. The guarantee depends on details:

- Is the first image verified by firmware?
- Can it load an unsigned second stage?
- Can the second stage replace its policy through an unsigned configuration?
- Is the kernel authenticated?
- Is initramfs authenticated with the kernel or loaded as mutable external data?
- Can the verified kernel load an arbitrary module or write its own memory through a debug interface?
- Who can enroll new keys?
- Does a recovery path bypass the normal chain?

"Secure Boot is enabled" is the beginning of an audit, not its conclusion.

Other attackers lead to other requirements. A remote attacker without physical or administrative access may care more about kernel vulnerabilities than ESP tampering. An enterprise may need remote attestation proving a fleet booted approved measurements. A personal owner may want to run self-built kernels while excluding everyone else's. State the attacker, assets, and recovery assumptions first.

---

## 2. A chain of trust

A chain of trust begins with a component whose integrity is established outside the chain. Platform hardware and immutable or protected firmware code form a root of trust. That code validates the next component using a protected key or digest. The validated component becomes responsible for validating another, and so on.

A simplified Linux chain can be:

~~~text
hardware root / protected firmware
    -> UEFI firmware volumes
    -> shim EFI image
    -> GRUB or a Unified Kernel Image
    -> Linux kernel and initramfs
    -> kernel modules and privileged kernel extensions
    -> verified or authenticated root userspace
~~~

Not every system implements every arrow. Standard UEFI Secure Boot focuses on EFI image validation. Extending trust beyond that point is the responsibility of the loaded components and operating-system policy.

The chain is only as strong as its weakest accepted path. If the normal entry is strict but a fallback EFI file starts an unrestricted shell, physical modification can select the fallback. If firmware allows legacy CSM boot, raw sector code can bypass EFI image validation. If setup access permits disabling Secure Boot without authentication, physical possession changes the threat model.

---

## 3. UEFI Secure Boot variables

UEFI defines authenticated variables and signature databases that control image verification.

### Platform Key: PK

The Platform Key represents platform ownership. Installing a PK normally moves the platform out of Setup Mode into User Mode. The holder can authorize changes to key-exchange policy. Removing the PK can return the platform to Setup Mode, where key enrollment behavior changes.

### Key Exchange Keys: KEK

KEKs authorize updates to allowed and forbidden signature databases. A platform can trust multiple organizations to update policy, for example a hardware vendor and an operating-system ecosystem.

### Allowed database: db

The db variable contains trusted signatures, certificates, or hashes used to authorize images. If an EFI executable is correctly signed by a trusted certificate and not forbidden, firmware may execute it under policy.

### Forbidden database: dbx

The dbx variable revokes known-vulnerable images, certificates, or hashes. Revocation matters because a correctly signed old loader can contain a vulnerability that allows arbitrary code execution. Trust is not only "was it ever signed" but "is it still acceptable."

### Additional state

Variables such as SecureBoot and SetupMode expose current state. The specification also describes Audit Mode and Deployed Mode behavior. Firmware setup interfaces use vendor language that may not map cleanly to the formal state names, so inspect variables and documentation rather than relying on a colored toggle.

On Linux, a common read-only status check is:

~~~sh
mokutil --sb-state
bootctl status 2>/dev/null
~~~

The first tool reports platform state through EFI variables when available. It does not prove every later stage enforced validation.

---

## 4. Image validation

EFI executables use PE/COFF structures. Authenticode-style signatures cover defined portions of the image while excluding fields that must change during signing. Firmware parses the image, calculates its digest according to the specification, evaluates signatures and hashes against policy, and rejects forbidden or unauthorized images.

The verification result depends on more than a certificate printed by a tool:

- the signature must cover the relevant bytes correctly;
- the chain must lead to a key accepted by current firmware policy;
- algorithms and key sizes must be supported;
- the image or signer must not be revoked by dbx;
- timestamp or policy rules may affect acceptance;
- firmware implementation must be correct.

Offline inspection tools can list signatures:

~~~sh
sbverify --list /path/to/image.efi
pesign -S -i /path/to/image.efi 2>/dev/null
~~~

Tool availability varies. Listing a signature is not the same as simulating the exact firmware policy. The firmware's actual decision at boot is authoritative for that stage.

---

## 5. Why Linux distributions use shim

Many commodity systems ship with firmware that trusts a broadly deployed third-party signing ecosystem but does not contain each distribution's private root. Shim is a small first-stage EFI application designed to bridge that gap.

A distribution obtains a signature on shim that firmware accepts. Shim contains or recognizes the distribution's certificate and can validate the next component. It also supports owner-managed keys through the Machine Owner Key mechanism.

A common path is:

~~~text
UEFI db trusts shim signer
    -> firmware verifies shim
    -> shim trusts distribution certificate or enrolled MOK
    -> shim verifies GRUB
    -> GRUB verifies kernel or follows distribution policy
~~~

Shim also participates in revocation and may enforce restrictions when it detects Secure Boot mode. Its exact behavior depends on build, version, protocol state, and distribution integration.

The design narrows the firmware trust requirement: firmware needs to start shim, and shim implements Linux-oriented policy for later stages. It also creates another component that must be updated when vulnerabilities are found.

---

## 6. Machine Owner Keys

Machine Owner Keys let the local owner extend shim's trust. This is useful for self-built kernels or third-party kernel modules without replacing the platform PK/KEK/db hierarchy.

A typical enrollment flow is:

1. Generate or obtain a signing key and protect the private half.
2. Request enrollment of the public certificate through mokutil.
3. Reboot into the shim-managed enrollment interface.
4. Confirm the request with a one-time password or physical-console interaction.
5. Shim records the accepted key in a firmware variable it manages.
6. Later shim or kernel policy can trust objects signed by that key.

The reboot confirmation prevents any root process from silently enrolling permanent boot trust without a presence check—though a fully compromised root still has many powers, and platform threat models differ.

Never place the private signing key on every target ESP. Treat it like a release credential. For personal systems, an encrypted offline copy may be sufficient. For fleets, use controlled signing infrastructure, audit logs, rotation, and revocation plans.

Inspect enrolled MOK certificates without changing them:

~~~sh
mokutil --list-enrolled
~~~

Enrollment does not sign anything automatically. Images or modules must be signed and verification components must consult the corresponding keyring.

---

## 7. GRUB in a verified chain

Firmware or shim can authenticate the GRUB EFI binary, but GRUB is extensible. It may load modules, configuration, fonts, themes, environment blocks, kernels, and initramfs from storage.

A secure configuration must decide which of those inputs require authentication. If signed GRUB loads an unsigned module containing executable code, the chain ends at that module. If unrestricted GRUB commands can disable verification or read arbitrary configuration, a console attacker may redirect execution.

Distribution builds can lock down commands and require signatures under Secure Boot. Some embed essential modules and a small configuration into the signed image, then verify external content. Details are build-specific. Do not generalize from upstream capabilities to one installed grubx64.efi.

Questions for an audit:

- Was GRUB loaded directly or through shim?
- Which certificate signed it?
- Are external modules allowed, and must they be signed?
- Where is grub.cfg, and is it authenticated?
- Can a user edit the selected kernel command line?
- Does GRUB verify the kernel and initramfs separately?
- Can chainloading escape policy?
- Which rescue commands remain available?

Usability and recovery must be designed, not added as unsigned exceptions afterward.

---

## 8. Unified Kernel Images

A Unified Kernel Image packages related boot material into one EFI executable. Typical sections contain:

- an EFI stub;
- the Linux kernel;
- initramfs;
- an embedded command line;
- operating-system release metadata;
- optional Device Tree, splash, PCR signature, or hardware identifiers.

Signing one UKI can authenticate the kernel, early userspace, and command line as a unit. This closes a gap where a signed kernel consumes a malicious external initramfs. It also supports measured-boot policies tied to predictable sections.

The UKI does not make its contents confidential; anyone who can read the ESP can normally read or extract sections. It provides integrity and origin authentication when signature policy is enforced.

Operationally, create a new versioned UKI, verify its signature and contents, copy it atomically where practical, add a boot entry, test it, and retain a known-good image. Never overwrite the only bootable UKI before validation.

---

## 9. Kernel signature and lockdown

Once the kernel runs, maintaining the chain requires controlling ways that privileged users can alter kernel execution.

### Kernel image authentication

The kernel may have been verified by GRUB, shim, or the EFI stub path. Some architectures and mechanisms support appended kernel signatures. The exact verifier and format depend on distribution design.

### Module signing

Linux can verify PKCS#7 signatures appended to kernel modules. Build-time and command-line policy can make invalid or unsigned modules fail to load. Trusted keys are placed in kernel keyrings from built-in certificates and platform or secondary sources according to configuration.

Inspect policy and messages:

~~~sh
cat /proc/sys/kernel/tainted
cat /sys/module/module/parameters/sig_enforce 2>/dev/null
journalctl -b -k | grep -Ei 'secure boot|lockdown|module verification|integrity'
keyctl list %:.platform 2>/dev/null
~~~

Paths and keyring visibility depend on configuration.

### Lockdown

Kernel lockdown restricts interfaces that allow even root to read or modify kernel memory or execute arbitrary code paths, because a root process could otherwise bypass module verification. Restrictions may affect /dev/mem, kexec, hibernation, unsigned modules, debug facilities, and hardware access.

Lockdown is often automatically enabled when booted through an enforced Secure Boot path on distribution kernels. It is a policy layer, not a direct synonym for firmware Secure Boot.

---

## 10. The initramfs gap

A signed kernel that accepts an unauthenticated external initramfs can execute the archive's /init as root. Whether this is possible depends on the loading and verification design.

Possible solutions include:

- embed initramfs in a signed UKI;
- have a verified loader authenticate the initramfs;
- use a kernel-integrated or architecture-specific authenticated image format;
- make early userspace verify the next root and keep its own image inside the trusted boundary.

Simply storing initramfs next to a signed kernel does not cryptographically bind them. File ownership and FAT attributes on the ESP are not a trust boundary against offline modification.

The command line can create a similar gap. If an attacker can append init=/bin/sh, disable integrity checks, or redirect root to malicious storage, signed code may faithfully execute attacker-selected policy. Embed or authenticate security-critical parameters and restrict interactive editing when the threat model requires it.

---

## 11. Measured boot

Verified boot decides whether execution is allowed. Measured boot records what was encountered.

A Trusted Platform Module exposes Platform Configuration Registers, or PCRs. A PCR is normally updated through an extend operation:

~~~text
new_PCR = Hash(old_PCR || measurement)
~~~

This creates an order-sensitive accumulated value. Components measure firmware, option ROMs, Secure Boot policy, boot applications, configuration, kernel sections, or other state into defined PCRs. An event log records individual measurements so software can replay them and explain the final values.

The TPM can sign a quote over selected PCRs using an attestation key. A remote verifier compares the quote and event log against acceptable policy. Alternatively, a secret can be sealed so the TPM releases it only under approved PCR states.

Measurement does not automatically block a malicious component. If policy accepts every measurement, the system records compromise very accurately and still boots it. Verification and measurement are complementary.

Inspect available evidence with tools from tpm2-tools and systemd where configured:

~~~sh
systemd-analyze pcrs 2>/dev/null
tpm2_pcrread 2>/dev/null
ls /sys/kernel/security/tpm* 2>/dev/null
~~~

PCR allocation and event interpretation are platform-specific. Do not bind the only copy of a recovery key to one brittle measurement set.

---

## 12. Disk encryption

Full-disk or root-volume encryption protects confidentiality when storage is offline. Secure Boot protects code integrity along the boot path. Measured boot can bind key release to platform state.

Combined example:

1. Firmware verifies a signed UKI.
2. UKI sections are measured into PCRs.
3. Early userspace asks the TPM to unseal a LUKS key only if measurements match policy.
4. The decrypted root is mounted.

This reduces exposure to an evil-maid modification of boot files: changing the UKI either fails signature verification or produces different measurements, preventing automatic key release.

Recovery is essential. Firmware updates, dbx updates, kernel updates, command-line changes, and boot-manager changes can legitimately alter PCRs. A design needs signed PCR policies, authorized policy updates, or a recovery secret stored separately. Security that bricks the fleet on the next firmware rollout is not robust security.

---

## 13. IMA and fs-verity

The Integrity Measurement Architecture can measure or appraise files as the kernel accesses them. Appraisal can require valid signatures or hashes according to policy. fs-verity provides Merkle-tree-based read-only file authenticity for individual files when a trusted digest is available.

These mechanisms extend integrity into userspace, but they are not automatically enabled by Secure Boot. Policy determines which files are measured, appraised, or protected, and how keys enter trust keyrings.

dm-verity protects a block-device tree, commonly used for immutable root images. A trusted root hash must be authenticated by the earlier chain. If an attacker can change both the verity tree and an unauthenticated root hash on the command line, integrity collapses.

Again, the boundary question is decisive: where does the first trusted digest come from?

---

## 14. Revocation and the boot-hole problem

Signed software can be vulnerable. A loader bug may allow an attacker to escape verification and execute arbitrary code. Fixing the latest loader is not enough if firmware still accepts the old vulnerable signed binary from removable media.

Revocation databases solve part of this problem. dbx updates can forbid particular image hashes or signing certificates. The Secure Boot Advanced Targeting mechanism and vendor update processes help distribute revocations without breaking unrelated images.

Revocation is operationally delicate:

- Update replacement boot files before revoking old ones.
- Include fallback and recovery media in inventory.
- Confirm firmware has space and correctly updates variables.
- Test unusual architectures and old machines.
- Preserve a recovery path that remains trusted.

A dbx update is intentionally difficult to reverse under attack. Treat it like a platform-policy change, not an ordinary package update.

---

## 15. Key ownership models

### Vendor-managed default

Firmware contains common trust roots, a distribution uses a broadly trusted shim, and users receive seamless boot. This maximizes compatibility but trusts the external signing and revocation ecosystem.

### Owner-extended

Default keys remain, and the owner enrolls a MOK for custom kernels or modules. This is common and practical. It does not remove trust in existing db entries.

### Owner-controlled platform keys

The owner replaces PK, KEK, and db with private policy. This provides strong control but creates responsibility for signing every boot path, firmware utility, option ROM requirement, recovery medium, and key rotation. Firmware interfaces differ, and a mistake can make recovery difficult.

### Appliance-managed

An organization provisions keys, signs UKIs, disables interactive overrides, uses measured boot, and controls updates centrally. Recovery and hardware replacement procedures are part of the design.

No model is universally correct. Ownership, interoperability, physical access, and operational maturity decide.

---

## 16. Safe inspection lab

This lab changes nothing.

### Record platform state

~~~sh
mokutil --sb-state
cat /sys/firmware/efi/efivars/SecureBoot-* 2>/dev/null | hexdump -C
cat /sys/firmware/efi/efivars/SetupMode-* 2>/dev/null | hexdump -C
~~~

efivarfs files begin with attribute bytes, so raw output requires correct parsing. Prefer mokutil or a purpose-built reader for conclusions.

### Identify first-stage files

~~~sh
sudo efibootmgr -v
find /boot/efi/EFI /efi/EFI -maxdepth 3 -type f -name '*.efi' -print 2>/dev/null
~~~

### List signatures

~~~sh
find /boot/efi/EFI /efi/EFI -type f -name '*.efi' -print0 2>/dev/null |
while IFS= read -r -d '' image; do
    echo "== $image =="
    sbverify --list "$image" 2>/dev/null || echo 'no readable signature or tool unavailable'
done
~~~

### Inspect kernel policy

~~~sh
journalctl -b -k | grep -Ei 'secure boot|lockdown|integrity|verification|trusted key'
cat /sys/kernel/security/lockdown 2>/dev/null
~~~

Draw the observed chain and mark every edge as verified, measured, both, or unknown. Unknown is an honest result and a research task.

---

## 17. Common failure patterns

### Image works with Secure Boot off but not on

Inspect architecture, signature, trust chain, dbx revocation, shim policy, and external modules. Disabling Secure Boot is a useful comparison, not necessarily an acceptable repair.

### Custom module is rejected

The module may be unsigned, signed after modification, signed by a key not in a trusted kernel keyring, or built for the wrong kernel. Confirm module version and signature metadata, MOK enrollment, and enforcement messages.

### MOK enrollment appears to disappear

The request and the completed enrollment are separate. Confirm the reboot enrollment screen was completed, firmware variables persist, and shim reads the expected variable namespace.

### Firmware update changes TPM unlock

Measurements or Secure Boot variables may have changed. Use the event log and PCR policy to identify which value moved. Recover with the separately stored key, update authorized policy, and do not erase the encrypted volume.

### Signed kernel boots with malicious initramfs

The chain authenticated the kernel but not the external archive. Move the archive inside a signed UKI or add explicit verification and bind critical command-line policy.

---

## 18. Recovery is part of trust

A trusted boot design needs a trusted recovery design.

Keep:

- at least one tested, signed recovery image;
- offline copies of required public certificates and protected private keys;
- a disk-encryption recovery secret independent of TPM policy;
- documentation of firmware key-reset behavior;
- a way to restore NVRAM entries and ESP files;
- an out-of-band console for remote systems;
- versioned boot artifacts and rollback criteria;
- a plan for compromised-key revocation.

Recovery media that disables every integrity control may be acceptable under physical custody, but then physical custody is part of the security boundary. Write that assumption down.

---

## 19. Mechanism summary

UEFI Secure Boot uses authenticated platform variables and signature databases to decide which EFI images may execute. PK establishes platform ownership, KEKs authorize database updates, db allows images, and dbx revokes them. Linux distributions commonly use a firmware-trusted shim to connect platform trust to distribution and owner-managed keys.

The chain must continue. GRUB modules and configuration, the kernel, initramfs, command line, kernel modules, and integrity policy all matter. A UKI can bind kernel, initramfs, and parameters into one signed object. Kernel lockdown and module-signature enforcement reduce post-boot ways to bypass the chain.

Measured boot extends hashes into TPM PCRs and supports attestation or sealed secrets; it does not itself block execution. Disk encryption protects confidentiality; it does not authenticate the unlock program. Combining verification, measurement, encryption, and recoverable key policy produces a stronger system than treating any one checkbox as complete security.

The next chapter returns to execution. With the kernel image accepted and loaded, we will enter its smallest and most constrained code: the decompressor and architecture startup path.

## 20. Three threat-model walkthroughs

### Shared workstation with an untrusted visitor

The visitor can reboot the machine and attach removable media but cannot open the chassis or authenticate to firmware setup. The intended control is that firmware accepts only authorized EFI images, prevents legacy boot, protects changes to boot order and Secure Boot state, and starts a signed UKI whose embedded command line and initramfs cannot be replaced independently. Root storage is encrypted, and the unlock key is entered only after the trusted UKI has displayed a recognizable prompt.

Secure Boot reduces the chance that a fake unlock program captures the passphrase. It does not prevent the visitor from stealing the whole device, observing a password with a camera, exploiting a vulnerable signed loader, or attacking a running unlocked session. Chassis-opening and hardware replacement require tamper evidence or stronger physical controls.

### Remote server with an administrator account compromise

The attacker obtains userspace root but no signing key or platform-key access. Enforced module signatures and lockdown prevent simple insertion of an unsigned kernel module or direct /dev/mem patching. A read-only, verified root design can make persistent replacement harder. Measured boot lets fleet control detect an unexpected next reboot.

This does not make userspace root harmless. The attacker can read application secrets, change mutable data, abuse allowed kernel interfaces, and attack the signing or update service. Live response and credential rotation remain necessary. The boot chain protects the next start and constrains persistence; it does not undo compromise of a running authorized system.

### Managed appliance with unattended TPM unlock

The appliance uses a vendor-owned UEFI key hierarchy, versioned signed UKIs, no interactive command-line editing, a dm-verity root, and a LUKS key sealed to signed PCR policy. Updates install a new UKI and authorized PCR policy before switching a one-shot boot entry. A watchdog confirms health and rolls selection back if the new image fails.

The recovery design keeps an offline recovery key and signed rescue UKI. Without both, a legitimate firmware or revocation update could make the storage permanently inaccessible. The strongest part of this design is not a particular algorithm; it is the transactional relationship among image installation, signature validation, PCR policy authorization, selection, health confirmation, and rollback.

## 21. Key lifecycle checklist

A signing key is a lifecycle, not a file.

Document who creates it, where private material is stored, which build service can request signatures, how approvals are recorded, how targets receive public trust, when certificates expire, what happens after personnel changes, and how compromise triggers revocation. Keep test and production roots separate. A test key copied into production firmware can quietly turn a well-designed chain into a public signing oracle.

Inventory every accepted signer before rotation. Firmware db may trust more certificates than the organization remembers. Shim may trust built-in distribution keys and MOKs. The kernel may carry built-in module certificates plus platform-derived keys. One forgotten key can preserve an unintended path; one prematurely removed key can strand recovery media.

Practice rotation on a representative machine:

1. Install new trust while old trust still works.
2. Sign a new version with the new key.
3. Boot it through the exact production path.
4. Confirm fallback and rescue artifacts.
5. Revoke or remove old trust only after fleet evidence is complete.
6. Test a machine that missed intermediate updates.

The same staged logic applies to dbx revocations. Trust changes must be ordered so the platform always has at least one acceptable, nonvulnerable path.

---

## Primary references

- UEFI specifications and Secure Boot chapters: https://uefi.org/specifications
- UEFI Boot Manager variables: https://uefi.org/specs/UEFI/2.11/03_Boot_Manager.html
- shim source and documentation: https://github.com/rhboot/shim
- Linux module signing: https://docs.kernel.org/admin-guide/module-signing.html
- Linux IMA: https://docs.kernel.org/security/IMA-templates.html
- Linux kernel lockdown: https://man7.org/linux/man-pages/man7/kernel_lockdown.7.html
- Unified Kernel Image specification: https://uapi-group.org/specifications/specs/unified_kernel_image/
- TPM 2.0 Library Specification: https://trustedcomputinggroup.org/resource/tpm-library-specification/
- systemd TPM and measured-boot documentation: https://systemd.io/TPM2_PCR_MEASUREMENTS/
- fs-verity documentation: https://docs.kernel.org/filesystems/fsverity.html
- dm-verity documentation: https://docs.kernel.org/admin-guide/device-mapper/verity.html
