# Reverse engineering an image check of SNR-CPE-W4N-MT

[![Social](https://img.shields.io/badge/social-telegram-lightgray.svg)](https://teleg.run/c1ewd)
[![IDA Pro](https://img.shields.io/badge/IDA_Pro-7.5-orange.svg)](https://hex-rays.com)
[![Donations](https://img.shields.io/badge/donations-Liberapay-green.svg)](https://liberapay.com/c1ewd/donate)

> Reversing image header

When we ported OpenWRT to SNR-CPRE-W4N-MT router, we have a trouble with upload firmware over stock uploader. The stock uploader was sending `"bad header checksum!"` message to a terminal. When trying to fix it without deep analyse, we was getting often `"corrupted data!"` message.
First we needed to find what is sending this messages. And we found. The command below led us to cwmpd:

```
/ # grep -rn "has bad header checksum!" /bin
/bin/cwmpd:110128:*** ERROR: "%s" has bad header checksum!
```

Armed with a disassembler, we have been founded the section of code where this message occurs. The text is at `0x0044bd58` offset. It is referenced by the code from the function at `0x0003eab0` offset.

Function of check checksum below:

```
check_checksum:
lw      $v0, 0x114+var_F4($fp)
nop
sw      $v0, 0x114+var_E8($fp)
lw      $v0, 0x114+var_F4($fp)
nop
lw      $v0, 4($v0)
nop
move    $a0, $v0
la      $v0, ntohl
nop
move    $t9, $v0
jalr    $t9 ; ntohl
nop
lw      $gp, 0x114+var_FC($fp)
sw      $v0, 0x114+var_E4($fp)
move    $a0, $zero
la      $v0, htonl
nop
move    $t9, $v0
jalr    $t9 ; htonl
nop
lw      $gp, 0x114+var_FC($fp)
move    $v1, $v0
lw      $v0, 0x114+var_F4($fp)
nop
sw      $v1, 4($v0)
move    $a0, $zero
lw      $a1, 0x114+var_E8($fp)
li      $a2, 0x40  # '@'
jal     calc
nop
lw      $gp, 0x114+var_FC($fp)
move    $v1, $v0
lw      $v0, 0x114+var_E4($fp)
nop
beq     $v1, $v0, check_currupted
nop
```

The calc function directly calculates the checksum. An analysis of the calc function led to the conclusion that the calculation algorithm is a standard CRC32 calculation that roams the Internet. Argument `$a0` - initial checksum offset, `$a1` - pointer to the file, `$a2` - calculation area size (header size).

Calculation CRC32 over entire file:

```
check_currupted:
lw      $v0, 0x114+var_EC($fp)
nop
addiu   $v0, 0x40  # '@'
sw      $v0, 0x114+var_E8($fp)
lw      $v0, 0x114+arg_8($fp)
nop
addiu   $v0, -0x40
sw      $v0, 0x114+var_E0($fp)
lw      $v0, 0x114+var_E0($fp)
move    $a0, $zero
lw      $a1, 0x114+var_E8($fp)
move    $a2, $v0
jal     calc
nop
lw      $gp, 0x114+var_FC($fp)
move    $s0, $v0
lw      $v0, 0x114+var_F4($fp)
nop
lw      $v0, 0x18($v0)
nop
move    $a0, $v0
la      $v0, ntohl
nop
move    $t9, $v0
jalr    $t9 ; ntohl
nop
lw      $gp, 0x114+var_FC($fp)
beq     $s0, $v0, loc_43F0C4
nop
```

In this function arguments of calc are different from arguments in calc_checksum function. Here `$a1` - pointer to the file plus `0x40`, `$a2` - file size minus `0x40`.
As it turned out, the calculation of CRC32 goes over the entire area of the file minus the header. From this we can conclude that we are dealing with a non-standard uImage. Reversing map of the firmware header below:

```
Start   Size    Value(HEX)  Description
0x00    0x04    27051956    Image Header Magic Number
0x04    0x04    73D35107    Image Header (0x00 : 0x40) CRC32
0x08    0x04    570D1C14    Image Creation Timestamp
0x0C    0x04    00566AAE    File size - 0x40
0x10    0x04    80000000    Data  Load  Address
0x14    0x04    80247580    Entry Point Address
0x18    0x04    49599B6F    (0x40 : EOF) CRC32
0x1c    0x01    0x05        Operating System
0x1d    0x01    0x05        CPU architecture
0x1e    0x01    0x02        Image Type
0x1f    0x01    0x03        Compression Type
0x20    0x20    534E522D4350452D57344E2D4D5400000000000000000000000000000010473A Image Name(SNR-CPE-W4N-MT                 G:)
```

Unfortunately, OpenWRT convention says that "firmware" must be compatibility with uImage. And DTB should define compatible "denx,uimage" for it, then it's auto-splitted properly. Therefore we resorted to cunning and was installing the firmware in 2 steps. First, over the stock upload initramfs. Second, over initramfs upload sysupgrade.

## Donations (Optional)

[![Liberapay](https://liberapay.com/assets/widgets/donate.svg)](https://liberapay.com/c1ewd/donate)

