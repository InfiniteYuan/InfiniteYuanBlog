---
title: ESP32 官方文档（八）Flash 加密
date: 2018-09-02 00:13:51
categories:
- ESP32 官方文档
tags:
- ESP32
---

# Flash 加密

Flash 加密功能用于加密 ESP32 连接的 SPI  Flash 的内容。启用 Flash 加密后，通过物理方式读取 SPI Flash 的内容不足以恢复大多数 Flash 内容。

Flash 加密与安全启动功能是分离的，您可以使用 Flash 加密而无需启用[安全启动](https://docs.espressif.com/projects/esp-idf/en/latest/security/secure-boot.html)。但是，我们建议将这两种功能一起用于安全的环境。在没有安全启动的情况下，需要执行其他配置以确保 Flash 加密的有效性。有关更多详细信息，请参阅[使用无安全启动的 Flash 加密](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#flash-encryption-without-secure-boot).

> 启用闪存加密会限制您进一步更新 ESP32 的选项。请务必阅读本文档(包括 [Flash 加密限制](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#flash-encryption-limitations))并了解启用闪存加密的含义。

<!--more-->

## 1 背景

 - 使用带有 256 位密钥的 AES 加密闪存的内容。闪存加密密钥存储在芯片内部的 efuse 中，并且(默认情况下)受软件访问保护。
 - 通过 ESP32 的闪存缓存映射功能，Flash 访问是透明的 - 映射到地址空间的任何闪存区域在读取时都将被透明地解密。
 - 通过使用明文数据烧录 ESP32 来应用加密，并且(如果启用了加密)引导加载程序会在首次启动时对数据进行加密。
 - 并非所有闪存都是加密的。以下类型的闪存数据已加密:
	 - 引导程序
	 - 安全启动引导加载程序摘要(如果启用了安全启动)
	 - 分区表
	 - 所有 “app” 类型分区
	 - 分区表中标有 “encrypted” 标志的任何分区

		可能希望一些数据分区保持未加密以便于访问，或者使用对于数据被加密时无效的闪存友好更新算法。由于 NVS 库与闪存加密不直接兼容，因此无法加密用于非易失性存储的 NVS 分区。有关更多详细信息，请参阅 [NVS 加密](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/storage/nvs_flash.html#nvs-encryption)。
 - 闪存加密密钥存储在 ESP32 芯片内部的 efuse 密钥块 1 中。默认情况下，此密钥具有读写保护功能，因此软件无法访问或更改密钥。
 - 默认情况下，Efuse Block 1 编码方案为 “None”，并且在该块中存储 256 位密钥。在某些 ESP32 上，编码方案设置为 3/4 编码 (CODING_SCHEME efuse 的值为 1)，并且必须在该块中存储 192 位密钥。有关详细信息，请参见 《ESP32 技术参考手册》第 20.3.1.3 节 “系统参数 coding_scheme”。在所有情况下，算法都在 256 位密钥上运行，通过重复一些位（[具体细节](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#flash-encryption-algorithm)）来扩展  192 位密钥。当 `esptool.py` 连接到芯片或 `espefuse.py summary` 输出时，编码方案显示在 Features 行中。
 - 闪存加密算法是 AES-256，其中密钥是“调整”的，每个 32 字节闪存块的偏移地址。这意味着每个 32 字节块(两个连续的 16 字节 AES 块)使用从闪存加密密钥派生的唯一密钥进行加密。
 - 虽然芯片上运行的软件可以透明地解密闪存内容，但默认情况下，当启用闪存加密时，UART 引导加载程序无法解密(或加密)数据。
 - 如果可以启用闪存加密，则编写[使用加密闪存](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#using-encrypted-flash)的代码时，编程人员必须采取一定的预防措施。

## 2 Flash 加密初始化

这是默认(推荐) Flash 加密初始化过程。可以为开发或其他目的自定义此过程，有关详细信息，请参阅 [Flash 加密高级功能](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#flash-encryption-advanced-features)。

> 在首次启动时启用闪存加密后，硬件通过串行重新闪存最多允许3次后续闪存烧录。必须遵循特殊过程(在[串口烧录](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#updating-encrypted-flash-serial)中记录)才能执行这些更新。

 - 如果启用了安全启动，则使用纯文本数据进行物理重新刷新需要“可重新映射”的安全启动摘要(请参阅 [Flash 加密和安全启动](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#flash-encryption-and-secure-boot))。
 - OTA 更新可用于更新 Flash 内容，而不计入此限制。
 - 在开发中启用闪存加密时，请使用预生成的闪存加密密钥，以允许使用预加密数据进行无限次重新闪存。

启用闪存加密的过程:

 - 必须在启用闪存加密支持的情况下编译引导加载程序。在 `make menuconfig` 中，导航到“安全功能”，然后选择“是”以“启动时启用闪存加密”。
 - 如果同时启用安全启动，最好同时选择这些选项。首先阅读[安全启动](https://docs.espressif.com/projects/esp-idf/en/latest/security/secure-boot.html)文档。
 - 正常构建并刷新引导加载程序，分区表和工厂应用程序映像。这些分区最初写入未加密的闪存。

> 启用安全启动和闪存加密时，引导加载程序应用程序二进制 `bootloader.bin` 可能会变得太大。请参阅 [Bootloader 大小](https://docs.espressif.com/projects/esp-idf/en/latest/security/secure-boot.html#secure-boot-bootloader-size)。

 - 首次启动时，引导加载程序会看到 [FLASH_CRYPT_CNT efuse](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#flash-crypt-cnt) 设置为 0 (出厂默认值)，因此它使用硬件随机数生成器生成闪存加密密钥。该密钥存储在efuse中。密钥是读写保护，以防止进一步的软件访问。
 - 然后，引导加载程序就地对所有加密分区进行加密。就地加密可能需要一些时间(对于大型分区，最多需要一分钟.)

> 第一次启动加密通道运行时，请勿中断ESP32的电源。如果电源中断，闪存内容将被破坏，并且需要再次使用未加密的数据烧录。像这样的重新烧录不会计入烧录限制。

 - 烧录完成后。在 UART 引导加载程序运行时，激活(默认情况下)以禁用加密的闪存访问。有关高级详细信息，请参阅[启用 UART Bootloader 加密/解密](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#uart-bootloader-encryption)。
 - `FLASH_CRYPT_CONFIG efuse` 也会被烧制到最大值(0xF)，以最大化闪存算法中调整的关键位数。有关高级详细信息，请参阅[设置 FLASH_CRYPT_CONFIG](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#setting-flash-crypt-config)。
 - 最后，[FLASH_CRYPT_CNT efuse](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#flash-crypt-cnt) 以初始值 1 进行刻录。这个 efuse 激活透明闪存加密层，并限制后续重新刷新的次数。有关 [FLASH_CRYPT_CNT efuse](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#flash-crypt-cnt) 的详细信息，请参阅[更新加密的 Flash](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#updating-encrypted-flash) 部分。
 - 引导加载程序重置自身以重新加密的闪存重新引导。

## 3 使用加密的 Flash

ESP32 应用程序代码可以通过调用 `esp_flash_encryption_enabled()` 来检查当前是否启用了闪存加密。

启用闪存加密后，从代码访问闪存内容时需要注意一些事项。

### 3.1 Flash 加密的范围

只要将 [FLASH_CRYPT_CNT efuse](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#flash-crypt-cnt) 设置为设置了奇数位的值，就会透明地解密通过 MMU 的闪存缓存访问的所有闪存内容。这包括:

 - Flash 中的可执行应用程序代码 (IROM)。
 - 存储在闪存 (DROM) 中的所有只读数据。
 - 通过 `esp_spi_flash_mmap()` 访问的任何数据。
 - ROM 引导加载程序读取软件引导加载程序映像。

> MMU Flash 缓存无条件地解密所有数据。在闪存中未加密存储的数据将通过闪存缓存“透明地解密”，并且看起来像随机垃圾这样的软件。

### 3.2 读取加密的 Flash

要在不使用闪存缓存 MMU 映射的情况下读取数据，我们建议使用分区读取函数 `esp_partition_read()`。使用此功能时，只有从加密分区读取数据时才会解密数据。其他分区将以未加密方式读取。通过这种方式，软件可以以相同的方式访问加密和非加密的闪存。

通过其他 SPI 读取 APIs 读取的数据不会被解密:

 - 通过 `esp_spi_flash_read()` 读取的数据不会被解密
 - 通过 ROM 函数 `SPIRead()` 读取的数据不会被解密 (esp-idf 应用程序不支持此功能)
 - 使用非易失性存储 (NVS) API 存储的数据始终存储并读取解密。

### 3.3 写加密的 Flash

在可能的情况下，我们建议使用分区写入函数 `esp_partition_write`。使用此功能时，只有在写入加密分区时才会加密数据。数据将被写入未加密的其他分区。通过这种方式，软件可以以相同的方式访问加密和非加密的闪存.

当 `write_encrypted` 参数设置为 true 时，`esp_spi_flash_write` 函数将写入数据。否则，数据将以未加密的方式写入.

ROM 函数 `esp_rom_spiflash_write_encrypted` 将加密数据写入闪存，ROM 函数 `SPIWrite` 将未加密写入闪存。(esp-idf 应用程序不支持这些功能).

未加密数据的最小写入大小为 4 个字节(对齐为 4 个字节)。由于数据是以块为单位加密的，因此加密数据的最小写入大小为 16 字节(对齐为16字节).

## 4 更像加密的 Flash

### 4.1 OTA 更新

只要使用 `esp_partition_write` 函数，对加密分区的 OTA 更新将自动加密写入.

### 4.2 串口烧录

`FLASH_CRYPT_CNT efuse` 允许通过串口烧录(或其他物理方法)使用新的明文数据更新闪存，最多 3 次.

该过程涉及烧录明文数据，然后碰撞 `FLASH_CRYPT_CNT efuse` 的值，这会导致引导加载程序重新加密此数据.

#### 4.2.1 限制更新

这种类型只有 4 个明文串行更新周期，包括初始加密闪存.

禁用第四次加密后，`FLASH_CRYPT_CNT efuse` 的最大值为 0xFF，永久禁用加密.

通过预生成的 Flash 加密密钥使用 OTA 更新或重新刷新可以超过此限制.

#### 4.2.2 串口烧录的注意事项

 - 通过串口重新烧录时，重新刷新最初用明文数据写入的每个分区(包括 bootloader)。可以跳过不是“当前选择的” OTA 分区的应用程序分区(除非在那里找到明文应用程序映像，否则不会重新加密这些分区.)但是，标有“加密”标志的任何分区都将无条件地重新分区。加密，意味着任何已加密的数据将被加密两次并被破坏.
	 - 使用 `make flash` 应烧录所有需要闪存的分区.
 - 如果启用了安全启动，则除非您使用“可重新启动”选项进行安全启动，否则无法通过串口重新刷新纯文本数据。请参阅 [Flash加密和安全启动](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html#flash-encryption-and-secure-boot).

### 4.3 串口重新烧录程序

 - 正常的构建应用程序.
 - 正常的使用明文数据刷新设备 (make flash 或 esptool.py 命令.)闪存所有先前加密的分区，包括引导加载程序(参见上一节).
 - 此时，设备将无法启动(消息为 flash read err，1000)，因为它希望看到加密的引导加载程序，但引导加载程序是纯文本.
 - 通过运行命令 `espefuse.py burn_efuse FLASH_CRYPT_CNT` 来刻录 `FLASH_CRYPT_CNT efuse`。 `espefuse.py` 会自动将位数递增 1，从而禁用加密.
 - 重置设备，它将重新加密明文分区，然后再次刻录 `FLASH_CRYPT_CNT efuse` 以重新启用加密.

#### 4.3.1 禁用串口更新

要防止通过串口进行进一步的明文更新，请在启用闪存加密后(即首次启动完成后)使用 `espefuse.py` 写保护 `FLASH_CRYPT_CNT efuse` :

```
espefuse.py --port PORT write_protect_efuse FLASH_CRYPT_CNT
```

这可以防止进一步修改以禁用或重新启用闪存加密.

### 4.4 通过预生成的 Flash 加密密钥重新烧录

可以在主机上预生成闪存加密密钥，并将其刻录到 ESP32 的 efuse 密钥块中。这允许数据在主机上预加密并烧录到 ESP32，而无需明文闪存更新.

这对于开发很有用，因为它消除了 4 次刷新限制。它还允许在启用安全启动的情况下重新刷新应用程序，因为每次都不需要重新启动引导加载程序.

> 此方法仅用于协助开发，而不用于生产设备。如果为生产预生成闪存加密，请确保密钥是从高质量的随机数源生成的，并且不要跨多个设备共享相同的闪存加密密钥.

#### 4.4.1 预生成 Flash 加密密钥

Flash 加密密钥是 32 字节的随机数据。您可以使用 `espsecure.py` 生成随机密钥:

```
espsecure.py generate_flash_encryption_key my_flash_encryption_key.bin
```

(这些数据的随机性仅与操作系统一样好，而且是Python安装的随机数据源.)

或者，如果您使用安全启动并具有安全启动签名密钥，则可以生成安全启动专用签名密钥的确定性SHA-256摘要，并将其用作闪存加密密钥:

```
espsecure.py digest_private_key --keyfile secure_boot_signing_key.pem my_flash_encryption_key.bin
```

(如果为安全启动启用可重新映射模式，则使用相同的 32 个字节作为安全启动摘要键.)

以这种方式从安全启动签名密钥生成闪存加密密钥意味着您只需要存储一个密钥文件。然而，该方法根本不适用于生产设备.

#### 4.4.2 刻录 Flash 加密密钥

生成闪存加密密钥后，需要将其刻录到 ESP32 的 efuse 密钥块。这必须在首次加密启动之前完成，否则 ESP32 将生成软件无法访问或修改的随机密钥.

要将密钥刻录到设备(仅限一次):

```
espefuse.py --port PORT burn_key flash_encryption my_flash_encryption_key.bin
```

#### 4.4.3 带有预生成密钥的第一次烧录

烧录密钥后，按照与默认 Flash 加密初始化相同的步骤操作，并为第一次启动时刷新纯文本图像。引导加载程序将使用预先烧制的密钥启用闪存加密并加密所有分区.

#### 4.4.4 使用预生成密钥重新烧录

在首次启动时启用加密后，重新烧录加密镜像需要额外的手动步骤。这是我们预先加密我们希望在闪存中更新的数据的地方.

假设这是用于刷新明文数据的常规命令:

```
esptool.py --port /dev/ttyUSB0 --baud 115200 write_flash 0x10000 build/my-app.bin
```

二进制应用程序映像 build/my-app.bin 写入偏移量 0x10000。此文件名和偏移量需要用于加密数据，如下所示:

```
espsecure.py encrypt_flash_data --keyfile my_flash_encryption_key.bin --address 0x10000 -o build/my-app-encrypted.bin build/my-app.bin
```

此示例命令将使用提供的密钥加密 my-app.bin，并生成加密文件 my-app-encrypted.bin。确保 address 参数与计划闪存二进制文件的地址匹配.

然后，使用esptool.py刷新加密的二进制文件:

```
esptool.py --port /dev/ttyUSB0 --baud 115200 write_flash 0x10000 build/my-app-encrypted.bin
```

不需要进一步的步骤或 efuse 操作，因为我们闪存时数据已经加密.

## 5 禁用 Flash 加密

如果由于某种原因意外启用了闪存加密，则下一次明文数据闪存将使ESP32软化(设备将连续重启，打印错误闪存读错误， 1000).

您可以通过编写 `FLASH_CRYPT_CNT efuse` 再次禁用闪存加密:

 - 首先，运行 `make menuconfig` 并取消选中“安全功能”下的“启用闪存加密启动”.
 - 退出 `menuconfig` 并保存新配置.
 - 再次运行 `make menuconfig` 并仔细检查你是否真的禁用了这个选项！ 如果启用此选项，则引导加载程序将在引导时立即重新启用加密.
 - 运行 `make flash` 以构建并刷新新的引导加载程序和应用程序，而不启用闪存加密.
 - 运行 `espefuse.py` (在 components/esptool_py/esptool 中)以禁用 `FLASH_CRYPT_CNT efuse`)::

重置 ESP32 并禁用闪存加密，引导加载程序将正常启动.

## 6 Flash 加密的局限性

Flash 加密可防止加密闪存的明文读出，从而保护固件免受未经授权的读取和修改.了解闪存加密系统的局限性非常重要:

 - Flash 加密仅与密钥一样强大.因此，我们建议在首次启动时在设备上生成密钥(默认行为).如果在设备外生成密钥(请参阅通过预生成的 Flash 加密密钥重新刷新)，请确保遵循正确的步骤.
 - 并非所有数据都是加密存储的.如果在闪存上存储数据，请检查您使用的方法(库，API 等)是否支持闪存加密.
 - Flash 加密不会阻止攻击者理解闪存的高级布局.这是因为相同的 AES 密钥用于每对相邻的 16 字节 AES 块.当这些相邻的 16 字节块包含相同的内容(例如空或填充区域)时，这些块将加密以产生匹配的加密块对.这可能允许攻击者在加密设备之间进行高级别比较(即判断两个设备是否可能运行相同的固件版本).
 - 出于同样的原因，攻击者总能知道一对相邻的 16 字节块 (32 字节对齐)何时包含相同的内容.如果将敏感数据存储在闪存中，请记住这一点，设计闪存存储器，以免发生这种情况(使用计数器字节或每 16 字节一些其他不相同的值就足够了).

## 7 Flash 加密和安全启动

建议一起使用闪存加密和安全启动。但是，如果启用了安全启动，则重新刷新设备会有其他限制:

 - OTA 更新不受限制(前提是新应用程序使用安全启动签名密钥正确签名).
 - 只有选择了 Reflashable Secure Boot 模式并且预先生成安全启动密钥并将其刻录到 ESP32 (参见安全启动文档)，才能进行明文串口烧录更新。在此配置中， make bootloader 将生成预先消化的引导加载程序和安全引导摘要文件，以便在偏移量 0x0 处烧录。当遵循明文串行重新刷新步骤时，必须在烧录其他明文数据之前重新刷新该文件.
 - 如果未重新启动引导加载程序，仍可以通过预生成的 Flash 加密密钥重新刷新。重新刷新引导加载程序需要在安全引导配置中启用相同的 Reflashable 选项.

## 8 使用没有安全启动的 Flash 加密

如果在没有安全启动的情况下使用闪存加密，则可以使用串行重新烧录加载未经授权的代码。有关详细信息，请参阅串行烧录 然后，这个未授权的代码可以读取所有加密的分区(以解密的形式)，使闪存加密无效。这可以通过写保护 `FLASH_CRYPT_CNT efuse` 来避免，从而禁止串口重新烧录。FLASH_CRYPT_CNT可以使用命令对efuse进行写保护:

```
espefuse.py --port PORT write_protect_efuse FLASH_CRYPT_CNT
```

或者，应用程序可以在其启动过程中调用 `esp_flash_write_protect_crypt_cnt()`.

## 9 Flash 加密高级功能

以下信息对于高级使用闪存加密非常有用:

### 9.1 加密分区标志

某些分区默认是加密的。否则，可以将任何分区标记为需要加密:

在分区表描述 CSV 文件中，有一个标志字段.

通常留空，如果在此字段中写入 “encrypted” ，则分区将在分区表中标记为已加密，此处写入的数据将被视为已加密(与应用程序分区相同):

```
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,     0x9000,  0x6000
phy_init, data, phy,     0xf000,  0x1000
factory,  app,  factory, 0x10000, 1M
secret_data, 0x40, 0x01, 0x20000, 256K, encrypted
```

 - 默认分区表都不包含任何加密数据分区.
 - 没有必要将 “app” 分区标记为已加密，它们始终被视为已加密.
 - 如果未启用闪存加密，则“加密”标志不执行任何操作.
 - 如果您希望保护此数据不受物理访问读取或修改的影响，则可以将 phy_init 数据标记为可选的phy分区.
 - 无法将 nvs 分区标记为已加密.

### 9.2 启用 UART Bootloader 加密/解密

默认情况下，首次启动时，闪存加密过程将刻录 `DISABLE_DL_ENCRYPT` ， `DISABLE_DL_DECRYPT` 和 `DISABLE_DL_CACHE` :

在 UART 引导加载程序引导模式下运行时， `DISABLE_DL_ENCRYPT` 禁用闪存加密操作.
 `DISABLE_DL_DECRYPT` 在 UART 引导加载程序模式下运行时禁用透明闪存解密，即使 `FLASH_CRYPT_CNT efuse` 设置为在正常操作中启用它也是如此.
在 UART 引导加载程序模式下运行时， `DISABLE_DL_CACHE` 禁用整个 MMU 闪存缓存.
可以仅刻录其中一些 efuses，并在第一次引导之前对其余部分进行写保护(使用未设置值 0)，以便保留它们。例如:

```
espefuse.py --port PORT burn_efuse DISABLE_DL_DECRYPT
espefuse.py --port PORT write_protect_efuse DISABLE_DL_ENCRYPT
```

(注意，这些 efuse 中的所有 3 个都是通过一个写保护位禁用的，因此写保护将保护所有这些保护位.因此，在写保护之前必须设置任何位.)

> 由于 esptool.py 不支持写入或读取加密闪存，因此写保护这些 efuse 以保持它们不被设置目前不是非常有用.

> 如果未设置 `DISABLE_DL_DECRYPT(0)`， 这有效地使闪存加密无效，因为具有物理访问权限的攻击者可以使用UART引导加载程序模式(使用自定义存根代码)来读取闪存内容.

### 9.3 设置 `FLASH_CRYPT_CONFIG`

`FLASH_CRYPT_CONFIG efuse` 确定闪存加密密钥中用块偏移“调整”的位数。有关详细信息，请参阅 Flash 加密算法.

引导加载程序的首次引导始终将此值设置为最大 0xF.

可以手动编写这些 efuse， 并在首次启动之前写保护，以便选择不同的调整值。不建议这样做.

强烈建议在值为零时永远不要写保护 `FLASH_CRYPT_CONFIG`。如果此 efuse 设置为零，则不会调整闪存加密密钥中的任何位，并且闪存加密算法等同于 AES ECB 模式.

## 10 技术细节

以下部分提供有关闪存加密操作的一些参考信息.

### 10.1 `FLASH_CRYPT_CNT efuse`

`FLASH_CRYPT_CNT` 是一个 8 位 efuse 字段，用于控制闪存加密。Flash 加密根据此 efuse 中设置为 “1” 的位数启用或禁用:

 - 设置偶数位 (0,2,4,6,8) 时:禁用闪存加密，无法解密任何加密数据.
	 - 如果引导加载程序是使用“启动时启用闪存加密”构建的，那么它将看到这种情况并立即重新加密闪存，无论它何时找到未加密的数据.完成后，它会将 efuse 中的另一位设置为 “1”，这意味着现在设置了奇数个位.
		 - 在第一次纯文本引导时，位计数具有全新值 0，并且引导加载程序在加密后将其更改为位计数 1 (值 0x01).
		 - 在下一次明文闪存更新后，将位计数手动更新为 2 (值 0x03).重新加密引导加载程序后，将 efuse 位计数更改为 3 (值 0x07).
		 - 在下一个明文闪存之后，将位计数手动更新为 4 (值 0x0F).重新加密引导加载程序后，将 efuse 位计数更改为 5 (值 0x1F).
		 - 在最后的明文闪存之后，位计数被手动更新为 6 (值 0x3F).重新加密引导加载程序后，将 efuse 位计数更改为7 (值 0x7F).
 - 设置奇数位 (1,3,5,7) 时:启用透明读取加密闪存.
 - 设置完所有 8 位后(efuse 值 0xFF):禁用透明读取加密闪存，永久无法访问任何加密数据。Bootloader 通常会检测到这种情况并停止.为避免使用此状态加载未经授权的代码，必须使用安全引导或 `FLASH_CRYPT_CNT efuse` 必须写保护.

### 10.2 Flash 加密算法

 - AES-256 以 16 字节数据块运行.闪存加密引擎以 32 字节块，两个串联的 AES 块加密和解密数据.
 - AES 算法在闪存加密中反转使用，因此闪存加密“加密”操作是 AES 解密，“解密”操作是AES加密.这是出于性能原因，并未改变算法的有效性.
 - 主闪存加密密钥存储在 efuse(BLOCK1) 中，默认情况下不受进一步写入或软件读取的影响.
 - 每个 32 字节块(两个相邻的 16 字节 AES 块)使用唯一密钥加密.密钥源自efuse中的主闪存加密密钥，与闪存中该块的偏移量进行异或(“键调整”).
 - 具体的调整取决于 `FLASH_CRYPT_CONFIG efuse` 的设置.这是一个 4 位 efuse， 其中每个位都能对特定范围的关键位进行异或运算:

	 - 位 1， 该值的 0-66 位被异或.
	 - 位 2， 该值的 67-131 位被异或.
	 - 位 3， 该值的 132-194 被异或.
	 - 位 4， 该值的 195-256 位被异或.
	建议始终保留 `FLASH_CRYPT_CONFIG` 以设置默认值 0xF，以便所有关键位与块偏移进行异或.有关详细信息，请参阅设置 `FLASH_CRYPT_CONFIG`.
 - 块偏移的高 19 位(第 5 位到第 23 位)与主闪存加密密钥进行异或.选择此范围有两个原因:最大闪存大小为 16MB(24 位)，每个块为 32 字节，因此最低有效 5 位 始终为零.
 - 从 19 个块偏移位中的每一个到闪存加密密钥的 256 位存在特定映射，以确定哪个位与哪个位进行异或.请参阅 espsecure.py 源代码中的变量 `_FLASH_ENCRYPTION_TWEAK_PATTERN` 以获取完整的映射.
 - 要查看 Python 中实现的完整闪存加密算法，请参阅 espsecure.py 源代码中的 `_flash_encryption_operation()` 函数.

## 11 参考资料

 - [原文链接](https://docs.espressif.com/projects/esp-idf/en/latest/security/flash-encryption.html)
