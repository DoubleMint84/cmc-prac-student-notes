# Динамическая память
Маст-хэв для любого прогера на Си.

Чтобы познать дзен, **подключаем библиотеку stdlib.h**: `#include <stdlib.h>`

### Самая важная функция:

```c
void *malloc(size_t size);
// Выделение size байт памяти, получаем указатель на выделенную память
```

Нужно помнить, что malloc медленно работает, и поэтому эту функцию следует вызывать как можно реже!

_Правило этикета_: перед тем, как использовать полученный указатель, необходимо проверить, не указывает ли он на NULL.

## Пример

Следует всегда явно указывать тип перед вызовом маллока:

```c
int *a = (int *)malloc(20 * sizeof(int)); 
// количество элементов в массиве * размер типа, который вам нужен
```

## Джанго освобожденный, или освобождение памяти

**!!!!!!!!!!!** Программист __всегда__ должен освобождать используемую им память **в конце программы** или после того, как она стала ему не нужна:

```c
void free(void *p);
// Да не освободил он память в конце программы
``` 

### Таким образом, free - Самая важная функция 2.

##Реаллокация памяти
Если прогеру не хватило выделенной памяти(не хватает маны), то он может реаллоцировать еще больше памяти:

```c
void *realloc(void *p, size_t size);
```

Пример:

```c
int *a malloc(20 * sizeof(int)); 
// раньше в массиве было всего лишь 20 элементов...
...
a = realloc(40 * sizeof(int)); // ...а теперь у него в массиве 40 элементов!
// при этом первые 20 элементов не изменились!
// Аккуратнее с возможной утечкой памяти! 
// Если realloc не находит места в памяти, 
// то он в a вернет 0, а указатель будет утерян!

// вот таким нехитрым образом можно из реаллока получить malloc...
realloc(NULL, 10);
// ...и free! 
realloc(a, 0);
// НО ЗАЧЕМ?!

// Хотя это может кому-то пригодиться :()
```

## Челленджи)
### Задача 1: считать с потока ввода последовательность чисел(int), которая оканчивается -1. Вернуть указатель на массив.

```c
// Вроде бы идеальное решение, но есть нюанс...
int *f(void) {
    int *a = NULL;
    int count = 0, now;
    scanf("%d", &now);
    while (now != -1) {
        a = (int *)realloc(a, (count + 1) * sizeof(int));
        a[count++] = now;
        scanf("%d", &now);
    }
    return a;
}

// Соль в том, что malloc жрет много времени, 
// поэтому для оптимальной работы нужно выделять память вот так:

int *sf(void) {
    int *a = NULL;
    int count = 0, now;
    unsigned n = 0;
    scanf("%d", &now);
    while (now != -1) {
        if (count == n) {
            if (n == 0) {
                n = 1;
            } else {
                n <<= 1;
            }
            a = (int *)realloc(a, n * sizeof(int));
        }
        a[count++] = now;
        scanf("%d", &now);
    }
    return a;
}

// PS: автор данного куска кода понимает, что можно было бы 
// написать лаконичнее и красивее, но ему лень
``` 

## Каллок, или причем тут коллоквиум

```c
void *calloc(size_t nmemb, size_t size);
// Не спрашивайте, зачем это нужно

// Хотя скажу: это тупо дублер malloc
// nmemb = количество элементов, 
// size = размер необходимого типа 
// То есть умножение в malloc заменяется на calloc
```

## Мемципиай) и другие мемные функции

```c
void memcpy(void *dst, const void *src, size_t n);
// копирует данные из источника src в приемник dst, в n - количество байт
// Прогер ОБЯЗАН контролировать размеры src и dst!
// В memcpy НЕЛЬЗЯ передавать пересекающиеся области памяти! Undefined Behaviour

void memmove(void *src, const void *dst, size_t n);
// Безопасная версия memcpy (проверяет на пересечение областей памяти)
// На случай, если прогеру лень делать проверку

void memset(void *p, int c, size_t n);
// Заполняет память p байтом c




// PS: "Memes. The DNA of the soul!"
```

### Задача 2: Реализовать функцию memcpy.

```c
void memcpy(void *dst, const void* src, size_t n) {
    for (int i = 0; i < n; i++) {
        char s = *(((char *)(src)) + i);
        char *pd + ((char *) dst) + i;
        *pd = s; 
    }
}
```

# Двумерные массивы
Такие же массивы, только *многомерные*.

```c
int a[10][20];
// Выделяется 10 кусков по 20 блоков типа int. Всего 200 элементов.
```

```c
(a + i)[j] = a[i][j] = (a + i * sizeof(int) * 20 + j * sizeof(int))
```

### Это все, что нужно знать про многомерные массивы.
Хотя нет:

```c
int(*a)[20] = malloc(sizeof(int) * 20 + 10) // тип - указатель на массив int[20]

int foo(int a[][20]) или int foo(int *a[20]) или int foo(int **a)
```

## VLA-массивы
Появилось сие чудо в C99

```c
void f(int n, int m) {
    int a[n][m];
}

void f(int n, int m, int a[n][m]) // если массив будет раньше размера указан,
// то компилятор сделает вам атата!
```