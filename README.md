# Скрипт создания пользователя в AD
Скрипт по созданию, редактированию и отключению пользователей в AD. Работает с выгрузками из ЗУПа УПР и ЗУПа РЕГЛ. Выгрузки настроены со стороны 1С на dc-01-scr-01 в папки:
 - user_info - файлы выгрузки .csv
 - photos - файлы выгрузки .jpg

## Логика работы:
 1. Все необработанные за последние 20 дней выгрузки перемещаем в папку old
 2. Получаем список актуальных выгрузок с шаблоном вида:
    - [Полное ФИО]\_[yyyy-MM-dd события]\_\_[uid сотрудника]\_[uid изменения]\_[время выгрузки]
3. Разделяем все выгрузки на 3 типа:
    - Выгрузки с .csv и с .jpg
    - Выгрузки только с .csv 
    - Выгрузки только с .jpg
4. Обрабатываем выгрузки, в которых есть .csv: 
    - Получаем содержание каждого csv файла: 
        - $User_UID - ID сотрудника в 1C = extensionAttribute6 в учётной записи AD пользователя
        - $Full_Name - ФИО сотрудника 
        - $Working - Флаг, указывающий работает сотрудник или увольняется. 1 = принятие/изменения с работающим сотрудником. 0 = увольнение.
        - $Position - Наименование должности. 
        - $Position_uid - ID должности в 1С = extensionAttribute10 в учётной записи AD сотрудника и = extensionAttribute6 в учётной записи AD группе-шаблоне прав должности
        - $Department - Наименование отдела
        - $Department_uid - ID отдела в 1C = extensionAttribute11 в учётной записи AD сотрудника и = extensionAttribute6 в учётной записи AD группе-шаблоне прав отдела
        - $Company - Юридическое лицо
        - $Mobile_phone - Номер телефона
        - $Birthday - Дата рождения
        - $Location - Адрес работы = StreetAddress в учётной записи AD сотрудника
        - $Manager - ФИО руководителя 
        - $Manager_UID - ID руководителя в 1С = extensionAttribute6 в учётной записи AD руководителя 
        - $Request - Флаг, указывающий, будут ли изменения в учётной записи проводится через портал Bitrix. 1 - ожидаем заявку, 0 - изменения вносятся без заявки на портале
    - Обрабатываем полученные данные: 
        - Получаем отдельно имя, фамилию и отчество
        - Обрезаем наименования должности и отдела под формат AD - 64 символа
        - Получаем учётную запись в AD руководителя
        - Приводим телефонный номер под формат 7xxxxxxxxxx
        - Производим поиск учётной записи в AD сотрудника
        - Очищаем переменную сбора ошибок при отработке скрипта
    - Внесение изменений в AD согласно полученным данным: 
        - Проверка, должен ли работать сотрудник. По работающим сотрудникам: 
            - Проверка, ожидаем ли заявку. По выгрузкам, в которых ожидаем заявку: 
                - [Поиск заявки на портале - функция Get-Ticket](#функция-Get-Ticket)
                - Проверка наличия заявки. При наличии заявки: 
                    - Проверка, нашёлся ли пользователь в AD. При наличии пользователя в AD: 
                        - Проверка, включена ли учётная запись. При включённой учётной записи:
                          - #### Блок изменений параметров учётной записи AD:
                           - Проверка, отличаются ли должность, отдел или их ID в выгрузке и в учётной запипси AD сотрудника. Если отличаются: 
                             * Блок переводов. [Функция изменения прав учётной записи - Remove-Add-user-from-AD-groups](#функция-Remove-Add-user-from-AD-groups)
                           - [Функция изменения аттрибутов учётной записи - Update-User-AD-info](#функция-Update-User-AD-info)
                           - [Функция перемещения пользователя в другую OU - OU-User-Change](#функция-OU-User-Change)
                           - [Функция проверки аккаунта в Zoom - Check-Zoom](#функция-Check-Zoom)
                           - Переопределение переменной стандартного пароля на строку "Текущий"
                           - Назначение переменной для переименовывания файлов выгрузки на "_UserUpdated"
                        - При выключенной учётной записи: 
                          - #### Блок восстановления учётной записи в AD
                            - Включение учётной записи в AD
                            - Удаление учётной записи из группы disabled_users
                            - Перемещение во временную OU из OU disabled
                            - Установка стандартного пароля 
                            - [Функция создания и проверки домашней папки пользователя - Test-Profile-Folder](#функция-Test-Profile-Folder)
                            - [Функция изменения прав учётной записи - Remove-Add-user-from-AD-groups](#функция-Remove-Add-user-from-AD-groups)
                            - Добавление в группы по умолчанию
                            - [Функция изменения аттрибутов учётной записи - Update-User-AD-info](#функция-Update-User-AD-info)
                            - [Функция создания или редактирования почтового ящика - Create-Update-Mailbox](#функция-Create-Update-Mailbox)
                            - [Функция перемещения пользователя в другую OU - OU-User-Change](#функция-OU-User-Change)
                            - [Функция проверки аккаунта в Zoom - Check-Zoom](#функция-Check-Zoom)
                            - Назначение переменной для переименовывания файлов выгрузки на "_UserRestored"
                    - При отсутсвии пользователя в AD: 
                      - #### Блок создания новой учётной записи: 
                          * [Функция создания логина из ФИО - Get-AD-Login-From-Full_Name](#функция-Get-AD-Login-From-Full_Name)
                          * [Функция создания пользователя - Create-User-AD](#функция-Create-User-AD)
                          * [Функция создания и проверки домашней папки пользователя - Test-Profile-Folder](#функция-Test-Profile-Folder)
                          * [Функция изменения прав учётной записи - Remove-Add-user-from-AD-groups](#функция-Remove-Add-user-from-AD-groups)
                          * [Функция создания или редактирования почтового ящика - Create-Update-Mailbox](#функция-Create-Update-Mailbox)
                          * [Функция перемещения пользователя в другую OU - OU-User-Change](#функция-OU-User-Change)
                          * [Функция проверки аккаунта в Zoom - Check-Zoom](#функция-Check-Zoom)
                          * Назначение переменной для переименовывания файлов выгрузки на "_UserCreated"
                    - Функция обработки заявки на портале - Portal-Message
                    - Функция отправки информационного сообщения на телефон сотрудника - SMS-Sender
                    - Проверка отработки скрипта на наличие критических ошибок. При отсутсвии: 
                      - Функция переименовывания файлов выгрузки - Rename-Files
                    - При наличии ошибок: 
                      - Запись в лог списка всех ошибок, из-за которых переименовывания не произошло - по этим выгрузкам скрипт отработает ещё раз при следующем запуске
                - Если заявку на портале для пользователя не нашли:
                    - Запись в лог информации, о том, что по данной выгрузке заявки не обнаружено
            - По выгрузкам, в которых заявка не ожидается проводим схожие работы по пользователю, только без привязки к порталу:
                - Проверка, нашёлся ли пользователь в AD. При наличии пользователя в AD: 
                    - Проверка, включена ли учётная запись. При включённой учётной записи:
                        - [Блок изменений параметров учётной записи AD](#Блок-изменений-параметров-учётной-записи-AD)
                    - При выключенной учётной записи: 
                        - [Блок восстановления учётной записи в AD](#Блок-восстановления-учётной-записи-в-AD)
                - При отсутсвии пользователя в AD:
                    - [Блок создания новой учётной записи](#Блок-создания-новой-учётной-записи)
                - Проверка отработки скрипта на наличие критических ошибок. При отсутсвии: 
                    - Функция переименовывания файлов выгрузки - Rename-Files
                - При наличии ошибок: 
                    - Запись в лог списка всех ошибок, из-за которых переименовывания не произошло - по этим выгрузкам скрипт отработает ещё раз при следующем запуске
        - По неработающим сотрудникам: 
            - Проверка, нашёлся ли пользователь в AD. Если нашли: 
                - Отключение аккаунта, проверка на отключение. Если отключен: 
                    - Назначение переменной для переименовывания файлов выгрузки на "_UserDisabled"
                - Проверка на наличие у сотрудника учётной записи администратора. Если нашли: 
                    - Отключение учётной записи администратора
            - Если не нашли пользователя на отключение: 
                - Запись в переменную сбора ошибок информации о том, что пользователь не найден. 
            - Проверка отработки скрипта на наличие критических ошибок. При отсутсвии: 
                - Функция переименовывания файлов выгрузки - Rename-Files
            - При наличии ошибок: 
                - Запись в лог списка всех ошибок, из-за которых переименовывания не произошло - по этим выгрузкам скрипт отработает ещё раз при следующем запуске
5. Обрабатываем выгрузки, в которых нет .csv - только .jpg
    - Получаем информацию о пользователе через наименование файла: 
        - $User_UID - ID сотрудника в 1C = extensionAttribute6 в учётной записи AD пользователя
        - $Full_Name - ФИО сотрудника 
    - Производим поиск учётной записи в AD сотрудника
    - Очищаем переменную сбора ошибок при отработке скрипта
    - Проверка, нашёлся ли пользователь в AD. Если нашли:
        - Получаем наименование файла с расширением и передаём его в Python скрипт по обнаружению лица и обрезанию фото под размер AD
        - Меняем фото в учётной записи AD сотрудника 
        - Назначение переменной для переименовывания файлов выгрузки на "_UserUpdated"
    - Если пользователя не нашли: 
        - Запись в переменную сбора ошибок информации о том, что пользователь не найден.
     - Проверка отработки скрипта на наличие критических ошибок. При отсутсвии: 
        - Функция переименовывания файлов выгрузки - Rename-Files



## Функции 
### Функция Invoke-MySQLQuery 

### Функция Translit

### Функция Get-AD-Login-From-Full_Name

### Функция Get-Ticket

### Функция SMS-Sender

### Функция Portal-Message

### Функция Rename-Files

### Функция Create-User-AD

### Функция Test-Profile-Folder

### Функция Update-User-AD-info

### Функция OU-User-Change 

### Функция Check-Zoom

### Функция Create-Update-Mailbox

### Функция Remove-Add-user-from-AD-groups

### Функция Create-AD-Template
