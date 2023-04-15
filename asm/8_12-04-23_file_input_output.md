# 12.04.23 / Разбор КР и файловый ввод/вывод
## Разбор полетов на КР
### Задача 1 (вариант 1)
Даны два числа. Нужно узнать, на сколько бит нужно циклически сдвинуть первое число, чтобы получить второе число. Если это невозможно, вывести -1.
```nasm
extern io_get_udec, io_print_udec
section .text
global main
main:
    call io_get_udec
    mov EBX, EAX        ; 1-е число
    call io_get_udec
    mov EDX, EAX        ; 2-е число
    mov ECX, -1
    xor EAX, EAX
.loop:
    cmp EBX, EDX
    je .end
    rol EBX, 1
    inc EAX
    cmp EAX, 32
    cmove EAX, ECX
    je .end
.end:
    call io_print_гdec
    xor EAX, EAX
    ret
```

### Задача 1 (вариант 2)
Даны два числа. Нужно узнать, на сколько бит отличается первое число от второго.
```nasm
extern io_get_udec, io_print_udec
section .text
global main
main:
    call io_get_udec
    mov EBX, EAX        ; 1-е число
    call io_get_udec    ; 2-е число
    xor EBX, EAX
    xor EAX, EAX
    mov ECX, 31
.loop:
    mov ESI, 1
    and ESI, EBX
    add EAX, ESI
    shr EBX, 1
    loop .loop
.end:
    call io_print_udec
    xor EAX, EAX
    ret
```

### Задача 2
Дана непустая последовательность чисел, длина которой кратна 4 (или 5, в зависимости от варианта). Нужно найти 75%-процентиль (80%) последовательсти, то есть число, которое больше 3/4 (4/5) чисел массива.

```nasm
extern io_get_udec, io_print_udec
%define k 5
section .text
global main
main:
    mov EBP, ESP
    xor EBX, EBX    ; Длина массива
.input_loop:
    call io_get_udec
    cmp EAX, 0
    je .end_input
    push EAX
    inc EBX
    jmp .input_loop
.end_input:
    mov AX, BX
    mov DX, 0
    mov CX, k
    div CX
    movzx ECX, AX
    inc ECX ; ECX = length/k + 1
.outer_loop:
    mov EDX, [ESP] ; EDX = max
    mov ESI, 0 ; ESI = max index
    mov EDI, 0 ; EDI = current index
.inner_loop:
    mov EAX, [ESP + 4 * EDI]
    cmp EAX, EDX
    cmova EDX, EAX
    cmova ESI, EDI
    inc EDI
    cmp EDI, ECX
    jne .inner_loop
    mov dword[ESP + 4 * ESI], 0
    loop .outer_loop

    mov EAX, EDX
    call io_print_udec
    mov ESP, EBP
    xor EAX, EAX
    ret
```

### Задача 3
Даны два беззнаковых 16-битных числа, в которых зашифрованы полиномы. Нужно найти результат произведения полиномов и вывести этот резльтат в виде 32-битного беззнакового числа.

```nasm
extern io_get_udec, io_print_udec
section .text
global main
main:
    call io_get_udec
    mov bx, ax
    call io_get_udec
    mov dx, ax
    mov ECX, 0
    xor EAX, EAX
.loop:
    mov ESI, 1
    and SI, DX
    cmovnz SI, BX
    shl ESI, CL
    inc CL
    xor EAX, ESI
    shr DX, 1
    cmp CL, 16
    jne .loop
    call io_print_udec
    xor EAX, EAX
    ret
```

## Считывание и запись в обычных файлах (.txt)
В `input.txt` - последовательность целых чисел, оканчивающаяся нулем. Найти их сумму и записать ее в `output.txt`.

```nasm
section .rodata
    fw db "w", 0
    format db "%d", 0
    format2 db "%d", 0xA, 0
    inf db "input.txt", 0
    outf db "output.txt", 0
    fr db "r", 0
extern fprintf, fscanf, fopen, fclose

section .text
global main
main:
    enter
    and, ESP, ~15
    sub ESP, 16
    xor ESP, 16
    xor EBX, EBX
    mov dword[ESP], inf
    mov dword[ESP + 4]m fr
    call fopen
    mov ESI, EAX
.loop:
    mov dword[ESP], ESI
    mov dword[ESP + 4], format
    lea EAX, [ESP + 12] 
    mov dword[ESP + 8], EAX
    call fscanf
    mov EAX, dword[ESP + 12]
    test EAX, EAX
    jz .loop_end
    add EBX, EAX
    jmp .loop
.loop_end:
    mov dword[ESP], ESI
    call fclose
    mov dword[ESP], outf
    mov dword[ESP + 4], fw
    call fopen
    mov ESI, EAX
    mov dword[ESP], ESI
    mov dword[ESP + 4], format2
    mov dword[ESP + 8], EBX
    call fprintf
    mov dword[ESP], ESI
    call fclose
    leave
    xor EAX, EAX
    ret
```

## Работа с бинарными файлами
Считать из файла `data.bin` последовательность данных, соответствующих структуре:

```c
struct data {
    short tag;
    int value;
};

// Функция для считывания данных из бинарного файла
fread(void* buff, size_t size, size_t num, FILE* f)
```

```nasm
section .rodata
    name db "data.bin", 0
    rb db "rb", 0
extern fopen, fclose, fread, io_print_dec

section .text
global main
main:
    mov EBP, ESP
    and ESP, ~15
    sub ESP, 32
    mov dword[ESP], name
    mov dword[ESP + 4], rb
    call fopen
    mov dword[ESP + 28], EAX ; FILE*
.loop:
    lea EBX, [ESP + 20]
    mov dword[ESP], EBX
    mov dword[ESP + 4], 8
    mov dword[ESP + 8], 1
    mov EBX, [ESP + 28]
    mov dword[ESP + 12], EBX
    call fread
    cmp EAX, 1
    jne .end
    cmp word[ESP + 20], -273
    je .loop
.end:
    ; Завершить написание программы дается читателю в качестве простого упражнения.
```