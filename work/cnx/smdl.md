## SMDL Flow

### Source File

xlate: xigen.tsmdl(xls.tsmdl, xalu.tsmdl, media.tsmdl) -> xigen.smdl -> xigen_xx.pla -> xigen_xx.v

xls ld/st

xalu integr

media float

xiq_pair.smdl  控制发送到哪个decoder进行解析

### Include File for Input

xdefines source condition mapping the x86 inst

xopcode   1B/2B/3B 的 最后一个opcode

xgroup   group instruction

esc   x87相关的opcode

### Include File for Input

xrat    decode mapping to uop define

ratregs    select & reg index for dest & src reg

ratsizes define the uop size include dest size src size

ient    trap ucode address

cnasm     pram addr using in xlate decode,  segment descriptor & msr

smdl/fe_smdl

master_ratopcodes.smdl   uop opcode definition      unit-opcode

