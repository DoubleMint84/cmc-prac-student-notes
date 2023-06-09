# 01.03.23 / Продолжаем работать с условными переходами + метки inside
## Еще кое-что об условных переходах
### CMOVcc (conditional move)
Команда `CMOVcc` осуществляет проверку состояния регистра EFLAFS и производит операцию переноса данных, если соответствующие флаги находятся в состоянии, заданном командой. Если условие (cc) не выполняется, то продолжается исполнение программы с команды следующей за `CMOVcc`. Команды `CMOVcc` могут переносить 16- и 32-битные данные из памяти в регистр общего назначения или из одного регистра общего назначения в другой. Перенос 8-битных данных не поддерживается.

#### :bangbang: Важно
- Команды CMOVcc поддерживаются не всеми процессорами Pentium Pro, Pentium II ... Для определения того, поддерживает ли конкретная модель процессора указанную команду, следует использовать команду `CPUID`.

```nasm
CMOVcc op1, op2 ; Condition codes (cc) такие же, как и в Jcc
```

#### :floppy_disk: Операнды
- `op1` - r16, r32
- `op2` - r/m16, r/m32

### JCXZ, JECXZ, JNCXZ, JNECXZ (conditional jump - ECX/CX)

Командa `JCXZ` проверяет значение регистра `CX` и, если оно равно нулю, производится передача управления по адресу, заданному операндом команды.
Команда `JECXZ` производит аналогичные действия, но использует регистр `ECX`.

```nasm
	mov	cx, [di]
	jcxz l2 ; если cx = 0, то переходим к метке l2
	...
l2:	ret
```

## Задача №1, или gcc совсем обленился

```c
int a[10], b[10], n = 10;

for (int i = 0; i < n; i++) {
    b[i] = a[i];
}
```

Дан код на Си. **Вы знаете**, что делать.

```nasm
section .data
    n: dd 10

section .bss
    a: resd 10
    b: resd 10

section .text
global main
main:
        MOV ECX, dword[n]

.m1:    MOV EAX, dword[a + 4 * ECX - 4]
        MOV dword[b + 4 * ECX - 4], EAX
        DEC ECX
        JECXNZ .m1
.m2:
        nop
```

## Задача №2, или беды с "капиталом"

```c
char a[] = "hello";

for (int i = 0; a[i]; i++) {
    a[i] = a[i] + 'A' - 'a'
}
// По сути, эта программа переводит символы строки в верхний регистр.
// Обычно такие функции называют "капитализаторами" (но это не точно)
```

Дан код на Си. **Приступайте**.

```nasm
section .data
    a: db 'h', 'e', 'l', 'l', 'o', 0

section .text
global main
main:
        MOV EBX, 'A'
        SUB EBX, 'a'

        MOV EAX, a
        XOR ECX, ECX
.l:     mov CL, byte[EAX]
        JECXZ .end
        add byte[EAX], BL   ; можно вместо BL написать 'A' - 'a'
                            ; т. к. 'A' - 'a'  -  константное выражение
        INC EAX
        JMP .l
.end:
        nop
```

## Пришло время поговорить о метках
### Строение ассемблерной строки NASM
Как и в большинстве ассемблеров, каждая строка NASM содержит (если это не макрос, препроцессорная или ассемблерная директива) комбинацию четырех полей:
```nasm
    метка: инструкция операнд_1[, операнд_2[, ...]] ; комментарий
```

Как обычно, большинство этих полей необязательны; допускается присутствие или отсутствие любой комбинации метки, инструкции и комментария. Конечно, необходимость поля операндов определяется инструкцией процессора.

NASM не накладывает ограничений на количество пробелов в строке: метки могут иметь пробелы вначале, а инструкции могут не иметь никаких пробелов и т.п. Двоеточие после метки также необязательно. 

#### :bangbang: Во избежание ошибок обязательно ставим двоеточие!
- Если вы хотите поместить в строку инструкцию `lodsb`, а введете `lodab`, строка останется корректной, но вместо инструкции будет объявлена метка. 
- Выявить данные опечатки отчасти можно, введя в строке запуска NASM ключ `-w+orphan-labels` — в этом случае при обнаружении метки без заключительного двоеточия будет выдаваться предупреждение. 

Допустимыми символами в метках являются буквы, цифры, знаки `_`, `$`, `#`, `@`, `~`, `.` и `?`. Допустимые символы в начале метки (первый символ метки) — только буквы, точка `.` (**со специальным значением, об этом напишу ниже**), знак подчеркивания `_` и вопросительный знак `?`. 

### Чем эта точка так особенна?
NASM дает специальную трактовку символов, начинающихся с точки. Метка, начинающаяся с точки, обрабатывается как *локальная*. Это означает, что она неразрывно связана с предыдущей нелокальной меткой. Например:
```nasm
label1:    
    ; некоторый код 
.loop:     
    ; еще какой-то код 
    jne .loop 
    ret 
label2:    
    ; некоторый код 
.loop:     
    ; еще какой-то код  
    jne .loop 
    ret
```
В приведенном фрагменте каждая инструкция `JNE` переходит на строку непосредственно перед ней, т.к. два определения `.loop` остаются разделены в силу того, что каждое связано с предшествующей нелокальной меткой.

Данный способ обработки локальных меток позаимствован из ассемблера DevPac (Amiga); однако NASM делает шаг вперед — он позволяет обращаться к локальным меткам из другой части кода. Это достигается путем описания локальной метки на основе предыдущей нелокальной. Описания `.loop` в примере выше в действительности описывают два разных символа: `label1.loop` и `label2.loop`, поэтому если вам это действительно надо, то можете написать: 
```nasm
label3:
    ; некоторый код 
    ; и т.д. 
    jmp label1.loop
```
### One more thing...
Иногда бывает полезно, например, в макросах — определить метку, на которую можно ссылаться отовсюду, но которая не пересекается с обычным механизмом локальных меток. Такая метка не может быть нелокальной, так как существует последующее описание и ссылки на локальные метки; она также не может быть и локальной, вследствие того, что описывающий ее макрос не будет знать полное имя метки. Для разрешения этой проблемы в NASM введен третий тип меток, которые обычно используются только в описаниях макросов: если метка начинается со специального префикса ..@, она ничего не делает по отношению к механизму локальных меток. Таким образом, вы можете написать: 
```nasm
label1:   ; нелокальная метка 
.local:   ; это label1.local 
..@foo:   ; это специальный символ
label2:   ; другая нелокальная метка 
.local:   ; это label2.local 
    jmp ..@foo             ; переход на три строки вверх
```

## Задача №3, или наконец-то не перевод
Пусть:
`ECX` - количество элементов массива
`EBX` - адрес 1-го элемента массива
Требуется найти максимум в массиве `signed int` и записать его в `EDX`.

```nasm
; Решение
section .text
        MOV EDX, dword[EBX + 4 * ECX - 4]
        DEC ECX
.m1:    MOV EAX, dword[EBX + 4 * ECX - 4]
        CMP EAX, EDX
        CMOVG EDX, EAX
        DEC ECX
        JECXNZ .m1
```