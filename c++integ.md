

# Методические указания: Интеграция кода на C/C++ в проекты на Python 3

## 1. Цель работы

Научиться подключать существующие библиотеки на языках C/C++ к проектам на Python 3 с помощью стандартной библиотеки `ctypes`, компилировать исходный код в разделяемые библиотеки (`.so` / `.dll`), а также **расширять** низкоуровневый функционал высокоуровневыми возможностями Python (валидация типов, объектно-ориентированные обёртки, пакетная обработка данных).

---

## 2. Теоретические сведения

Язык Python часто используют как «клей» для соединения компонентов, написанных на более быстрых языках. Стандартный модуль `ctypes` позволяет загружать динамические библиотеки и вызывать экспортируемые из них функции без написания дополнительного компилируемого кода (например, без использования Python C API).

**Ключевые понятия:**

- **Разделяемая библиотека**: Файл с машинным кодом, который загружается в память процесса во время выполнения. В Linux/macOS имеет расширение `.so`, в Windows – `.dll`.
- **ABI (Application Binary Interface)**: Соглашение о том, как функции принимают аргументы и возвращают значения. В `ctypes` мы должны явно указать `argtypes` (типы аргументов) и `restype` (тип возвращаемого значения), чтобы Python корректно преобразовывал данные и избегал ошибок сегментации (segfault).
- **`extern "C"`**: Если библиотека написана на C++, компилятор искажает имена функций (name mangling) для поддержки перегрузки. Чтобы функция была видна из `ctypes`, её объявление в C++ должно быть обёрнуто в блок `extern "C"`.

---

## 3. Порядок выполнения работы

### Шаг 1. Подготовка исходного кода

В реальных open-source проектах (например, в репозитории [Martinsos/edlib](https://github.com/Martinsos/edlib), предоставляющем сверхбыстрое вычисление расстояния редактирования) код может состоять из десятков файлов и собираться через CMake. Для освоения базового принципа мы используем упрощённый, но полноценный однофайловый вариант алгоритма расстояния Левенштейна.

Создайте рабочую директорию и сохраните в ней файл `levenshtein.c`:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

/* Вычисление расстояния Левенштейна между двумя строками */
int levenshtein_distance(const char *s1, const char *s2) {
    int len1 = strlen(s1);
    int len2 = strlen(s2);
    
    // Выделение памяти под матрицу динамического программирования
    int **dp = (int **)malloc((len1 + 1) * sizeof(int *));
    for (int i = 0; i <= len1; i++) {
        dp[i] = (int *)malloc((len2 + 1) * sizeof(int));
    }
    
    // Инициализация граничных условий
    for (int i = 0; i <= len1; i++) dp[i][0] = i;
    for (int j = 0; j <= len2; j++) dp[0][j] = j;
    
    // Заполнение матрицы
    for (int i = 1; i <= len1; i++) {
        for (int j = 1; j <= len2; j++) {
            int cost = (s1[i-1] == s2[j-1]) ? 0 : 1;
            int min1 = dp[i-1][j] + 1;
            int min2 = dp[i][j-1] + 1;
            int min3 = dp[i-1][j-1] + cost;
            dp[i][j] = min1 < min2 ? (min1 < min3 ? min1 : min3) : (min2 < min3 ? min2 : min3);
        }
    }
    
    int result = dp[len1][len2];
    
    // Освобождение памяти
    for (int i = 0; i <= len1; i++) free(dp[i]);
    free(dp);
    
    return result;
}
```

### Шаг 2. Компиляция в разделяемую библиотеку

Откройте терминал в директории с файлом и выполните команду компиляции.

**Для Linux / macOS:**
```bash
gcc -shared -fPIC -o liblevenshtein.so levenshtein.c
```
*(Флаг `-fPIC` обязателен для создания позиционно-независимого кода, требуемого разделяемыми библиотеками).*

**Для Windows (при наличии MinGW/GCC):**
```bash
gcc -shared -o levenshtein.dll levenshtein.c
```

### Шаг 3. Базовый вызов из Python

Создайте файл `basic_call.py`. Мы загрузим библиотеку и вызовем функцию напрямую.

```python
import ctypes
import os

# 1. Определение пути к библиотеке (адаптируйте расширение под вашу ОС)
lib_name = "liblevenshtein.so" if os.name != "nt" else "levenshtein.dll"
lib_path = os.path.abspath(lib_name)

# 2. Загрузка библиотеки
lib = ctypes.CDLL(lib_path)

# 3. Настройка сигнатуры функции (КРИТИЧЕСКИ ВАЖНО для избежания segfault)
lib.levenshtein_distance.argtypes = [ctypes.c_char_p, ctypes.c_char_p]
lib.levenshtein_distance.restype = ctypes.c_int

# 4. Вызов функции (строки должны быть закодированы в байты)
str1 = b"kitten"
str2 = b"sitting"
distance = lib.levenshtein_distance(str1, str2)

print(f"Расстояние между '{str1.decode()}' и '{str2.decode()}': {distance}")
```

Запустите скрипт: `python3 basic_call.py`. Убедитесь, что вывод равен `3`.

### Шаг 4. Расширение функционала средствами Python (Основная задача)

«Голый» вызов `ctypes` неудобен: он требует ручного кодирования строк в `bytes`, не проверяет типы и не умеет работать со списками. **Задача студента** – дополнить C-код высокоуровневым Python-функционалом.

Создайте файл `advanced_wrapper.py`:

```python
import ctypes
import os
from typing import List, Tuple

class LevenshteinMatcher:
    """
    Python-обёртка, расширяющая функционал C-библиотеки.
    C отвечает за скорость вычислений, Python - за удобство и безопасность.
    """
    
    def __init__(self, lib_path: str):
        self.lib = ctypes.CDLL(lib_path)
        # Настраиваем сигнатуры один раз при инициализации
        self.lib.levenshtein_distance.argtypes = [ctypes.c_char_p, ctypes.c_char_p]
        self.lib.levenshtein_distance.restype = ctypes.c_int

    def get_distance(self, s1: str, s2: str) -> int:
        """Безопасный вызов с проверкой типов и кодированием в UTF-8"""
        if not isinstance(s1, str) or not isinstance(s2, str):
            raise TypeError("Оба аргумента должны быть строками (str)")
        
        # Кодируем строки для передачи в C
        b_s1 = s1.encode('utf-8')
        b_s2 = s2.encode('utf-8')
        return self.lib.levenshtein_distance(b_s1, b_s2)

    def find_best_match(self, target: str, candidates: List[str]) -> Tuple[str, int]:
        """
        НОВЫЙ ФУНКЦИОНАЛ НА PYTHON:
        C-библиотека умеет сравнивать только две строки. 
        Мы дополняем её возможностью поиска лучшего совпадения в списке.
        """
        if not candidates:
            raise ValueError("Список кандидатов не может быть пустым")
            
        best_match = None
        min_dist = float('inf')
        
        for candidate in candidates:
            dist = self.get_distance(target, candidate)
            if dist < min_dist:
                min_dist = dist
                best_match = candidate
                
        return best_match, min_dist

# === Демонстрация работы ===
if __name__ == "__main__":
    lib_name = "liblevenshtein.so" if os.name != "nt" else "levenshtein.dll"
    matcher = LevenshteinMatcher(os.path.abspath(lib_name))
    
    # 1. Безопасный вызов с поддержкой Unicode (UTF-8)
    dist = matcher.get_distance("привет", "превет")
    print(f"Расстояние: {dist}")
    
    # 2. Использование нового функционала (пакетная обработка)
    words = ["яблоко", "груша", "яблочко", "апельсин"]
    target_word = "яблок"
    
    best, distance = matcher.find_best_match(target_word, words)
    print(f"Лучшее совпадение для '{target_word}': '{best}' (расстояние: {distance})")
```

---

## 4. Содержание отчёта

1. Листинг исходного C-кода и команда его компиляции.
2. Листинг Python-кода с комментариями, объясняющими, какой именно функционал был *добавлен* на стороне Python (например, валидация типов, поддержка Unicode, работа со списками).
3. Скриншот или текстовый лог вывода работы программы `advanced_wrapper.py`.
4. Ответы на контрольные вопросы.

---

## 5. Контрольные вопросы

1. Почему при вызове функции из `ctypes` необходимо явно указывать `argtypes` и `restype`? Что произойдёт, если этого не сделать при передаче 64-битных указателей на 64-битной системе?
2. В чём разница между компиляцией статической библиотеки (`.a` / `.lib`) и разделяемой библиотеки (`.so` / `.dll`) в контексте использования с Python?
3. Как модифицировать объявление функции в C++ файле, чтобы она была корректно экспортирована и видна модулю `ctypes`? 
4. Какие архитектурные преимущества даёт подход, при котором «тяжёлые» вычисления остаются на C, а логика обработки коллекций и исключений пишется на Python?

---

## 6. Полезные ссылки

- [Официальная документация Python: Модуль ctypes](https://docs.python.org/3/library/ctypes.html)
- [Wrapping a C library in Python with ctypes (IPython Books)](https://ipython-books.github.io/54-wrapping-a-c-library-in-python-with-ctypes/)
- Реальный open-source проект, использующий этот паттерн: [Martinsos/edlib](https://github.com/Martinsos/edlib) (Лёгкая и быстрая C/C++ библиотека для вычисления расстояния редактирования, имеющая Python-обёртку).

---


