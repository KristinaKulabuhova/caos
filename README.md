## Сигналы
**Сигнал** - это механизм (межпроцессорного взаимодействия) передачи коротких сообщений (номер сигнала), как правило, прерывающий работу процесса, которому он был отправлен.

**Процесс** - некоторая сущность, которая обладает своей выделенной виртуальной памятью. Процесс запускается с помощью системного вызова fork.
Завершение работы процессора:

* Добровольное:
  - выход из функции main
  - вызоы функции exit
  - системный вызов -exit
 
 (Подходит для детерминированного выполнения: у программы есть начало и конец)
 
* Принудительное - отправкой сигнала:
  - команда *kill*
  - команда *killall*
  - запуск через *timeout*
  - выключение или перезагрузка
  - закрытие вкладки терминала
  - кнопочки Ctrl+C

Из-за чего возникают сигналы:
1. Нарушение сигментации
2. запись в закрытый канал или сокет
3. деление на ноль
4. недопустимая инструкция
5. нарушение assertion

Сигналы могут быть посланы процессу:

* ядром, как правило, в случае критической ошибки выполнения;
* другим процессом, если есть у него права (запущены из одного пользователя, либо пользователь root);
* самому себе.

Номера сигналов начинаются с 1. Значение 0 имеет специальное назначение (см. ниже про `kill`). Некоторым номерам сигналов соответствуют стандартные для `POSIX` названия и назначения, которые подробно описаны man `7 signal`.

При получении сигнала процесс может:

* Игнорировать его. Это возможно для всех сигналов, кроме `SIGSTOP` и `SIGKILL`.
* Обработать отдельной функцией. Кроме `SIGSTOP` и `SIGKILL`
* Выполнить действие по умолчанию, предусмотренное назначением стандартного сигнала POSIX. Как правило, это завершение работы процесса.
* Изменить состояние: sTopped | Running

По умолчанию, все сигналы, кроме `SIGCHILD` (информирование о завершении дочернего процесса) и `SIGURG` (информировании о поступлении TCP-сегмента с приоритетными данными), приводят к завершению работы процесса.

Если процесс был завершён с помощью сигнала, а не с ипользованием системного вызова `exit` то для него считается не определенным код возврата. Родительский процесс может отследить эту ситуацию, используюя макросы `WIFSIGNALED` и `WTERMSIG`
```
pid_t child = ...
...
int status;
waitpid(child, &status, 0);
if (WIFEXITED(status)) {
    // дочерний процесс был завершён через exit
    int code = WEXITSTATUS(status); // код возврата
}
if (WIFSIGNALED(status)) {
    // дочерний процесс был завершёл сигналом
    int signum = WTERMSIG(status); // номер сигнала
}
```
Отправить сигнал любому процессу можно с помощью команды `kill` По умолчанию отправляется сигнал `SIGTERM` но можно указать в качестве опции, какой именно сигнал нужно отправить. Кроме того, некоторые сигналы отправляются терминалом, например *Ctrl+C* посылает сигнал `SIGINT` а Ctrl+\ - сигнал `SIGQUIT`

Сигналы:
|  Номер  |  Имя   | По умолчанию  |     Описание     |
| :---:   | :----: | :-----------: | :--------------: |
|    1    | SIGHUP |     Term      | обрыв соединения |
|    2    | SIGINT |     Term      |      Ctrl+C      |
|    3    | SIGQUIT|     Core      |      Ctrl+\      |
|    4    | SIGILL |     Core      | плохая инструкция (посылается ядром)|
|    6    | SIGABRT|     Core      |     abort()      |
|    9    | SIGKILL|     Term      |     убийство     |
|    11   | SIGSEGV|     Core      |что-то плохое с памятью|
|    13   | SIGPIPE|     Term      |Broken pipe|
|    15   | SIGTERM|     Term      |завершение работы (можно перехватить)|
|    17   | SIGCHILD|     Ign      |заверение доч. процесса|
|    18   | SIGCONT|     Cont      |Команда fg -возобновление процесса|
|    19   | SIGSTOP|     Stop      |Ctrl+Z|
|    23   | SIGURG|     Ign      |Socket urgent data|

### Пользовательские сигналы
Изначально в POSIX было зарезервировано два номера сигнала, которые можно было использовать на умотрение пользователя: `SIGUSR1` и `SIGUSR2`.

Кроме того, в Linux предусмотрен диапазон сигналов с номерами от `SIGRTMIN` до `SIGRTMAX`, которые можно использовать на усмотрение пользователя.

Действием по умолчанию для всех "пользовательских" сигналов является завершение работы процесса.

### Отправка сигналов программным способом
#### Системный вызов `kill`
По аналогии с одноимённой командой, kill предназначен для отправки сигнала любому процессу.
```
int kill(pid_t pid, int signum); // возврашает 0 или -1, если ошибка
```
Отправлять сигналы можно только тем процессам, которые принадлежат тому пользователю, что и пользователь, по которым выполняется системный вызов `kill`. Исключение составляет пользователь `root`, который может всё. При попытке отправить сигнал процессу другого пользователя, `kill` вернёт значение `-1`.

Номер процесса может быть меньше `1` в случаях:

* `0` - отправить сигнал всем процессам текущей группы процессов;
* `-1` - отправить сигнал всем процессам пользователя (использовать с осторожностью!);
* отрицательное значение -PID - отправить сигнал всем процессам группы PID.
Номер сигнала может принимать значение `0`, - в этом случае никакой сигнал не будет отправлен, а `kill` вернёт значение `0` в том случае, если процесс (группа) с указанным `pid` существует, и есть права на отправку сигналов.

### Функции `raise` и `abort`
Функция `raise` предназначен для отправки сигнала процессом самому себе. Функция стандартной библиотеки `abort` посылает самому себе сигнал `SIGABRT`, и часто используется для генерации исключительных ситуаций, которые получилось диагностировать во время выполнения, например, функцией `assert`.

### Системный вызов `alarm`
Системный вызов `alarm` запускает таймер, по истечении которого процесс сам себе отправит сигнал `SIGALRM`.
```
unsigned int alarm(unsigned int seconds);
```
Отменить ранее установленный таймер можно, вызвав `alarm` с параметром `0`. Возвращаемым значением является количество секунд предыдущего установленного таймера.

### Обработка сигналов
Сигналы, которые можно перехватить, то есть все, кроме `SIGSTOP` и `SIGKILL`, можно обработать программным способом. Для этого необходимо зарегистрировать функцию-обработчик сигнала.

### Системный вызов `signal`
```
#include <signal.h>

// Этот тип определен только в Linux!
typedef void (*sighandler_t)(int);

sighandler_t signal(int signum, sighandler_t handler); // для Linux
void (*signal(int signum, void (*func)(int))) (int); // по стандарту POSIX
```

```
void handler(int signum) {
  ...
}

int main() {
  signal(SIGINT, handler);
}
```
Системный вызов `signal` предназначен для того, чтобы зарегистрировать функцию в качестве обработчика определенного сигнала. Первым аргументом является номер сигнала, вторым - указатель на функцию, которая принимает единственный аргумент - номер пришедшего сигнала (т.е. одну функцию можно использовать сразу для нескольких сигналов), и ничего не возвращает.

Два специальных значения функции-обработчика `SIG_DFL` и `SIG_IGN` предназанчены для указания обработчика по умолчанию (т.е. отмены ранее зарегистрированного обработчика) и установки игнорирования сигнала.

Системный вызов `signal` возвращает указатель на ранее установленный обработчик.

### Отличия BSD от System-V:

* В System-V обработчик сигнала выполяется один раз, после чего сбрасывается на обработчик по умолчанию, а в BSD - остается неизменным.
* В BSD обработчик сигнала не будет вызван, если в это время уже выполняется обработчик того же самого сигнала, а в System-V это возможно.
* В System-V блокирующие системные вызовы (например, read) завершают свою работу при поступлении сигнала, а в BSD большинство блокирующих системных вызовов возобновляют свою работу после того, как обработчик сигнала заверщает свою работу.
* 
По этой причине, системный вызов `signal` считается устаревшим, и в новом коде использовать его запрещено, за исключением двух ситуаций:
```
signal(signum, SIG_DFL); // сброс на обработчик по умолчанию
signal(signum, SIG_IGN); // игнорирование сигнала
```

### Системный вызов `sigaction`
Системный вызов `sigaction`, в отличии от `signal`, в качестве второго аргумента принимает не указатель на функцию, а указатель на структуру `struct sigaction`, с которой, помимо указателя на функцию, хранится дополнительная информация, описывающая семантику обработки сигнала. Поведение обработчиков, зарегистрированных с помощью `sigaction`, не зависит от операционной системы.
```
int sigaction(int signum,
              const struct sigaction *restrict act,
              struct sigaction *oldact);
```

```
struct sigaction {
    void      (*sa_handler)(int);
    void      (*sa_sigaction)(int, siginfo_t*, void*);
    sigset_t  sa_mask;
    int       sa_flags;
    void      (*sa_restorer)(void);
};
```
Третьим аргументов является указатель на структуру, описывающую обработчик, который был зарегистрирован для этого. Если эта информация не нужна, то можно передать значение `NULL`.

Основные поля структуры struct sigaction:

* `sa_handler` - указатель на функцию-обработчик с одним аргументом типа `int`, могут быть использованы значения `SIG_DFL` и `SIG_IGN`;
* `sa_flags` - набор флагов, опиывающих поведение обработчика;
* `sa_sigaction` - указатель на функцию-обработчик с тремя параметрами, а не одним (используется, если в флагах присутствует `SA_SIGINFO`).

Некоторые флаги, которые можно передавать в `sa_flags`:

* `SA_RESTART` - продолжать выполнение прерванных системных вызовов (семантика BSD) после завершения обработки сигнала. По умолчанию (если флаг отсутствует) используется семантика System-V.
* `SA_SIGINFO` - вместо функции из sa_handler нужно использовать функцию с тремя параметрами int signum, siginfo_t *info, void *context, которой помимо номера сигнала, передается дополнительная информация (например PID отправителя) и пользовательский контекст.
* `SA_RESETHAND` - после выполнения обработчика сбросить на обработчик по умолчанию (семантика System-V). По умолчанию (если флаг отсутствует) используется семантика BSD.
8 `SA_NODEFER` - при повторном приходе сигнала во время выполени обработчика он будет обработан немедленно (семантика System-V). По умолчанию (если флаг отсутствует) используется семантика BSD.
