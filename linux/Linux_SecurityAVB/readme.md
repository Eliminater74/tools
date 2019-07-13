# **AVB Reference**

Release version: 2.2

Author email: jason.zhu@rock-chips.com

Date: 2019.06.03

Document confidentiality level: public information

------

## **Foreword**

   Suitable for RK3308 / RK3326 / PX30 / RK3399 / RK3328.

## **Overview**

## **product version**

| **Chip Name** | **Kernel Version** |
| -------- | -------- |
| RK3308 | All Kernel Versions |
| RK3326 | Kernel 4.4 |
| RK3399 | Kernel 4.4 |
| RK3328 | Kernel 4.4 |

## **Reader object**

This document (this guide) is primarily intended for the following engineers:

- Technical Support Engineer
- Software Development Engineer

## **Revision record**

** Date** | **Version** | **Author** | **Modification Description** |
| ---------- | ------ | --------- | --------------------- ------- |
| 2018.05.28 | V1.0 | jason.zhu | |
2018.06.12 | V1.1 | jason.zhu | Add Authenticated Unlock and some instructions |
2018.06.27 | V1.2 | Zain.Wong | Add key generation description |
| 2018.11.16 | V1.3 | Zain.Wong | Added uboot configuration instructions, compatible with rk3326 |
| 2019.01.28 | v2.0 | Zain.Wong | Synchronous uboot changes, add 3399 configuration |
| 2019.05.22 | v2.1 | Zain.Wong | Added 3328 support and distinguishes OTP/EFUSE avb programming commands |
| 2019.06.03 | v2.2 | Zain.Wong | Fix some inappropriate instructions |

[TOC]

## 1 . Notes

About device lock & unlock

  When the device is in the unlock state, the program will still verify the entire boot.img. If there is an error in the firmware, the program will report what is wrong, ** normal boot device**. If the device is in the lock state, the program will verify the entire boot.img. If the firmware is wrong, the next level of firmware will not be started. Therefore, the device is in the unlock state during the debugging phase, which is convenient for debugging.
  Once the device handles the lock state, Authenticated Unlock is required. For details, see 5. avb lock & unlock.

## 2 . Firmware Configuration

2.1. trust

Enter rkbin/RKTRUST, take rk3308 as an example, find RK3308TRUST.ini, modify

```
[BL32_OPTION]
SEC=0
Changed to
[BL32_OPTION]
SEC=1
```

2.2. uboot

 Uboot requires fastboot and optee support.

```
CONFIG_OPTEE_CLIENT=y
CONFIG_OPTEE_V1=y #rk312x/rk322x/rk3288/rk3228H/rk3368/rk3399 Mutually exclusive with V2
CONFIG_OPTEE_V2=y #rk3308/rk3326 Mutually exclusive with V1
```

Avb open needs to be configured in the config file

```
CONFIG_AVB_LIBAVB=y
CONFIG_AVB_LIBAVB_AB=y
CONFIG_AVB_LIBAVB_ATX=y
CONFIG_AVB_LIBAVB_USER=y
CONFIG_RK_AVB_LIBAVB_USER=y
CONFIG_AVB_VBMETA_PUBLIC_KEY_VALIDATE=y
CONFIG_CRYPTO_ROCKCHIP=y
CONFIG_ANDROID_AVB=y
CONFIG_ANDROID_AB=y # need to open again
CONFIG_OPTEE_ALWAYS_USE_SECURITY_PARTITION=y #rpmbOpen when not available, not open by default
CONFIG_ROCKCHIP_PRELOADER_PUB_KEY=y #efuse security scheme needs to be opened
```

Firmware, certificate and hash need to be programmed by fastboot, so you need to configure it in the config file.

```
CONFIG_FASTBOOT=y
CONFIG_FASTBOOT_BUF_ADDR=0x800800 #Different chip platforms
CONFIG_FASTBOOT_BUF_SIZE=0x04000000 #different chip platforms
CONFIG_FASTBOOT_FLASH=y
CONFIG_FASTBOOT_FLASH_MMC_DEV=0
```

Use ./make.sh xxxx to generate uboot.img, trust.img, loader.bin

2.3. parameter

AVB needs to add a vbmeta partition to store firmware signature information. Size 1M, position independent.

AVB needs the system partition. On the buildroot, rootfs partition, you need to rename rootfs to system. If you use uuid, modify the uuid partition name.

If the storage medium uses flash, you need to add another security partition to store the operation information. The content is encrypted and stored. Size 4M, position independent. Emmc does not need to add this partition, emmc operation information is stored in the physical rpmb partition

The following is an example of avb parameter:
~~~
0x00002000@0x00004000(uboot), 0x00002000@0x00006000(trust), 0x00002000@0x00008000(misc), 0x00010000@0x0000a000(boot), 0x00010000@0x0001a000(recovery), 0x00010000@0x0002a000(backup), 0x00020000@0x0003a000(oem), 0x00300000 @0x0005a000(system), 0x00000800@0x0035a000(vbmeta), 0x00002000@0x0035a800(security), -@0x0035c800(userdata:grow)
~~~

Avb ab parameter:
~~~
0x00002000@0x00004000(uboot), 0x00002000@0x00006000(trust_a), 0x00002000@0x00008000(trust_b), 0x00002000@0x0000a000(misc), 0x00010000@0x0000c000(boot_a), 0x00010000@0x0001c000(boot_b), 0x00010000@0x0002c000(backup), 0x00020000 @0x0003c000(oem), 0x00300000@0x0005c000(system_a), 0x00300000@0x0035c000(system_b), 0x00000800@0x0065c000(vbmeta_a), 0x00000800@0x0065c800(vbmeta_b), 0x00002000@0x0065d000(security), -@0x0065f00(userdata:grow)
~~~

When downloading, the name on the tool should be modified synchronously. After modification, the parameter is reloaded.

## 3 . Key

The AVB contains the following 4 keys:
Product RootKey (PRK): avb's root key
ProductIntermediate Key (PIK): intermediate key, mediation
ProductSigning Key (PSK): the key used to sign the firmware
ProductUnlock Key (PUK): used to unlock the device

** There is already a set of test certificates and keys in this directory. If you need new keys and certificates, you can generate them yourself by following the steps below.
Please keep the generated files in a safe place, otherwise you will not be able to unlock them after locking, and the machine will not be able to flash. **

~~~
    Openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -outform PEM -out testkey_prk.pem
    Openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -outform PEM -out testkey_psk.pem
    Openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -outform PEM -out testkey_pik.pem
    Touch temp.bin
    Python avbtool make_atx_certificate --output=pik_certificate.bin --subject=temp.bin --subject_key=testkey_pik.pem --subject_is_intermediate_authority --subject_key_version 42 --authority_key=testkey_prk.pem
    Echo "RKXXXX_nnnnnnnn" > product_id.bin
    Python avbtool make_atx_certificate --output=psk_certificate.bin --subject=product_id.bin --subject_key=testkey_psk.pem --subject_key_version 42 --authority_key=testkey_pik.pem
    Python avbtool make_atx_metadata --output=metadata.bin --intermediate_key_certificate=pik_certificate.bin --product_key_certificate=psk_certificate.bin
~~~

Temp.bin needs to create a temporary file, create a new temp.bin, no need to fill in the data.
Product_id.bin needs to be defined by itself, accounting for 16 bytes, which can be defined as a product ID.
Permanent_attributes.bin generates:

~~~
    Python avbtool make_atx_permanent_attributes --output=permanent_attributes.bin --product_id=product_id.bin --root_authority_key=testkey_prk.pem
~~~

PUK generation:
~~~
    Openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -outform PEM -out testkey_puk.pem
~~~

Puk_certificate.bin permanent_attributes.bin is the certificate for the device to be unlocked.
The generation process requires the use of PrivateKey.pem, which is the key that is burned into efuse/otp.
(Refer to Rockchip-Secure-Boot-Application-Note-V1.9), the process is as follows:
~~~
    Python avbtool make_atx_certificate --out
