# Управление реестром Windows NT (`reg`)

Операционная система Wunix содержит нативную реализацию системного реестра **Windows NT Registry**, обеспечивающего хранение системных конфигураций, параметров оборудования и настроек приложений.

---

## 1. Структура и хранение реестра

В Wunix ульи реестра (Registry Hives) хранятся в специальном виртуальном каталоге:
```
/win/registry/
├── system.reg      # Улей HKEY_LOCAL_MACHINE\SYSTEM
├── software.reg    # Улей HKEY_LOCAL_MACHINE\SOFTWARE
└── user.reg        # Улей HKEY_CURRENT_USER
```

Файл `system.reg` использует стандартный текстовый формат `REGEDIT4 / Windows Registry Editor Version 5.00`:
```ini
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control]
"SystemStartOptions"="NOEXECUTE=OPTIN"

[HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System\CentralProcessor\0]
"ProcessorNameString"="Wunix Virtual CPU x86_64"
"Identifier"="Intel64 Family 6 Model 1 Stepping 1"
```

---

## 2. Утилита `reg`

Для взаимодействия с реестром из командной строки Wunix предоставляет утилиту `reg`.

### Чтение параметров (`reg query`)
Позволяет исследовать подразделы и значения ключей:
```bash
# Просмотр всего подраздела
reg query "HKLM\SYSTEM\CurrentControlSet\Control"

# Запрос конкретного параметра
reg query "HKLM\SYSTEM\CurrentControlSet\Control" /v "SystemStartOptions"
```

### Добавление и изменение параметров (`reg add`)
Позволяет создать ключ или изменить строковое/числовое значение:
```bash
# Добавление строкового значения REG_SZ
sudo reg add "HKLM\Software\Wunix" /v "Version" /t REG_SZ /d "0.1.0-alpha" /f

# Добавление 32-битного числа REG_DWORD
sudo reg add "HKLM\SYSTEM\CurrentControlSet\Services\Net" /v "Start" /t REG_DWORD /d 2 /f
```
Ключевые флаги:
* `/v ИМЯ` — имя параметра.
* `/t ТИП` — тип данных (`REG_SZ`, `REG_DWORD`, `REG_BINARY`, `REG_EXPAND_SZ`).
* `/d ЗНАЧЕНИЕ` — данные.
* `/f` — перезаписать существующее значение без запроса подтверждения.

### Удаление параметров и разделов (`reg delete`)
```bash
sudo reg delete "HKLM\Software\Wunix" /v "Version" /f
sudo reg delete "HKLM\Software\Wunix" /f
```

### Экспорт и импорт (`reg export` и `reg import`)
Администратор может сохранить ветку реестра в файл или импортировать готовый конфигурационный файл:
```bash
# Экспорт ветки в файл
reg export "HKLM\SYSTEM" /home/user/system_backup.reg

# Импорт конфигурации
sudo reg import /home/user/system_backup.reg
```
