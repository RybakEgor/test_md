# Скрипт создания пользователя в AD
Скрипт по созданию, редактированию и отключению пользователей в AD. Работает с выгрузками из ЗУПа УПР и ЗУПа РЕГЛ. Выгрузки настроены со стороны 1С на dc-01-scr-01 в папки:
 - user_info - файлы выгрузки .csv
 - photos - файлы выгрузки .jpg

## Логика работы:
1. Получение вводных данных с помощью [функции подключения к менеджеру паролей passwork - TeamPass_Debug](#функция-TeamPass_Debug) - логинов и паролей, необходимых для отработки скрипта, а именно:
   - Получение данных для подключения к порталу Bitrix
   - Получение данных для подключения к Zoom
   - Получение данных для отправки сообщений на мобильный телефон через gateway.api.sc
   - Получение данных для подключения к Exchange
   - Получения данных для подключения к БД Bitrix
3. Все необработанные за последние 20 дней выгрузки перемещаем в папку old
4. Получаем список актуальных выгрузок с шаблоном вида:
    - [Полное ФИО]\_[yyyy-MM-dd события]\_\_[uid сотрудника]\_[uid изменения]\_[время выгрузки]
5. Разделяем все выгрузки на 3 типа:
    - Выгрузки с .csv и с .jpg
    - Выгрузки только с .csv 
    - Выгрузки только с .jpg
6. Обрабатываем выгрузки, в которых есть .csv: 
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
        1. Проверка, должен ли работать сотрудник. По работающим сотрудникам:
            - Проверка, ожидаем ли заявку. По выгрузкам, в которых ожидаем заявку: 
                - [Поиск заявки на портале - функция Get-Ticket](#функция-Get-Ticket)
                - Проверка наличия заявки. При наличии заявки: 
                    - Проверка, нашёлся ли пользователь в AD. При наличии пользователя в AD: 
                        - Проверка, включена ли учётная запись. При включённой учётной записи:
                          - #### Блок изменений параметров учётной записи AD:
                           - Проверка, отличаются ли должность, отдел или их ID в выгрузке и в учётной запипси AD сотрудника. Если отличаются: 
                             * Блок переводов. [Функция изменения прав учётной записи - Remove-Add-user-from-AD-groups](#функция-Remove-Add-user-from-AD-groups)
                           - [Функция изменения атрибутов учётной записи - Update-User-AD-info](#функция-Update-User-AD-info)
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
                            - [Функция изменения атрибутов учётной записи - Update-User-AD-info](#функция-Update-User-AD-info)
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
                    - [Функция обработки заявки на портале - Portal-Message](#функция-Portal-Message)
                    - [Функция отправки информационного сообщения на телефон сотрудника - SMS-Sender](#функция-SMS-Sender)
                    - Проверка отработки скрипта на наличие критических ошибок. При отсутсвии: 
                      - [Функция переименовывания файлов выгрузки - Rename-Files](#функция-Rename-Files)
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
                    - [Функция переименовывания файлов выгрузки - Rename-Files](#функция-Rename-Files)
                - При наличии ошибок: 
                    - Запись в лог списка всех ошибок, из-за которых переименовывания не произошло - по этим выгрузкам скрипт отработает ещё раз при следующем запуске
        2. По неработающим сотрудникам: 
            - Проверка, нашёлся ли пользователь в AD. Если нашли: 
                - Отключение аккаунта, проверка на отключение. Если отключен: 
                    - Назначение переменной для переименовывания файлов выгрузки на "_UserDisabled"
                - Проверка на наличие у сотрудника учётной записи администратора. Если нашли: 
                    - Отключение учётной записи администратора
            - Если не нашли пользователя на отключение: 
                - Запись в переменную сбора ошибок информации о том, что пользователь не найден. 
            - Проверка отработки скрипта на наличие критических ошибок. При отсутсвии: 
                - [Функция переименовывания файлов выгрузки - Rename-Files](#функция-Rename-Files)
            - При наличии ошибок: 
                - Запись в лог списка всех ошибок, из-за которых переименовывания не произошло - по этим выгрузкам скрипт отработает ещё раз при следующем запуске
7. Обрабатываем выгрузки, в которых нет .csv - только .jpg
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
        - [Функция переименовывания файлов выгрузки - Rename-Files](#функция-Rename-Files)


===================================================================
## Функции 

### Функция TeamPass_Debug
Коннектор подключения к менеджеру паролей passwork. Использует библиотеку maximumps

Функция получает на вход: 
 - Наименование папки
 - Наименование записи
 - Логин

Производит подключение к passwork через API, используя личный ключ API сервисной учётной записи, и поиск указанной в вводных данных записи. 
Результат работы функции - получение логина и пароля из менеджера паролей passwork

### Функция Invoke-MySQLQuery 
Коннектор подключения к базе данных SQL . Работает с использованием DLL MySQL Connector Net 8.0.12
Получает на вход строку подключения к базе данных SQL и сам запрос в базу данных.
Результатом отработки функции является результат запроса в БД, приведённый в виде таблицы. 

### Функция Translit
Функция транслитерации входной строки посимвольно. Используется для генерации логина пользователя в AD, а также для создания алиаса почтового ящика новых групп-шаблонов. 
#### Логика работы 
- Полученное на вход значение переводится в тип string 
- Из полученной строки создаётся массив букв
- Каждая буква транслитерируется. Любая русская буква переводится в английский аналог. Для корректного создания псевдонима для почты групп-шаблонов был добавлен английский алфавит и цифры - они остаются неизменными.
- Результатом отработки функции является транслитерированная строка

### Функция Get-AD-Login-From-Full_Name
Генерация уникального логина в AD по ФИО для новых сотрудников с проверкой на уникальность.

Результатом отработки функции является уникальный логин для сотрудника в AD

#### Логика работы
- Проверка, является ли ФИО сотрудника полным
- При наличии у сотрудника хотя бы имени и фамилии сформировывается логин на русском языке по формату
   - Первая буква имени + первая буква отчества + фамилия - при наличии полного ФИО
   - Первая буква имени + фамилия - при отсутсвии отчества
- При отсутсвии двух составляющих из фамилии , имени и отчества производим запись в переменную сбора ошибок информацию о том, что у пользователя некорректное ФИО
- Если логин на русском сформирован, то создаём логин для AD [функцией транслитерации - Translit](#функция-Translit)
  - Производим проверку на уникальность логина. Пока находим пользователя в AD с таким же логином, которым только что сформировали - продолжаем попытки унифицировать логин:
    1. При наличии полного ФИО:
      - Добавляем букву первого слова отчества в русский вариант логина
      - создаём логин для AD [функцией транслитерации - Translit](#функция-Translit)
      - Проводим проверку. Продолжаем пока не дойдём до уникального.
    2. При наличии только имени и фамилии:
      - добавляем букву имени
      - создаём логин для AD [функцией транслитерации - Translit](#функция-Translit)
      - Проводим проверку. Продолжаем пока не дойдём до уникального.
- Если логин не сформирован производим запись в переменную сбора ошибок информацию о том, что для пользователя не удалось создать уникальный логин

### Функция Get-Ticket
Функция поиска активной заявки с наименованием "Создание учётной записи в AD ФИО_сотрудника" на портале.

Результатом работы функции является номер заявки по созданию пользователя из выгрузки на портале.

#### Логика работы 
- Написан запрос в БД Bitrix на поиск всех активных заявок на портале по шаблону в наименовании заявки "Cоздание учетной записи в AD%".
- Актвными считаем все, кроме тех, у которых статус:
  1. 10 - на тестировании 
  2. 9 - удалено 
  3. 45 - выполнено
- Подключаемся к БД портала и передаём запрос с помощью [функции подключения к базе данных SQL - Invoke-MySQLQuery](#функция-Invoke-MySQLQuery)
- Получаем список всех активных заявок по созданию пользователя и ищем номер той, в которой наименование включает ФИО сотрудника из выгрузки
- При отсутсвии таких или наличии нескольких присваиваем переменной с номером заявки $null


### Функция SMS-Sender
Функция отправки сообщений на мобильный телефон пользователю. Отрабатыает только при восстановлениях и создании новых пользователей AD. При обычных переводах/изменениях атрибутов переменая $Password_standart переопределена на 'Текущий'.

Результатом работы функции является отправленное на мобильный телефон сообщение.
#### Логика работы 
- Проверка, включена ли отправка сообщений. Если включена:
  - Проверка, включает ли переменная стандартного пароля цифры и указан ли в выгрузке мобильный телефон.  Если оба условия выполнены:
    - Формирование текста сообщения для пользователя с данными для входа, требованиями к новому паролю, данными для входа в зум и номером ИТ
    - Отправка сообщения через gateway.api.sc



### Функция Portal-Message
Функция обработки заявки на портале. Для вариаций отработки функции совершается сбор логов критических ошибок во всём скрипте. 

Результатом отработки функции является либо заявка переведённая на тестирование с информацией по пользователю из выгрузки, либо указание ошибок, из-за которых заявка не может быть закрыта. 

#### Логика работы
- Функция совершает конечные проверки отработки скрипта и в зависимости от наличия или отсутствия ошибок составляет скрытый или открытый комментария к заявке, а именно:
  - Проверка, создан ли пользователь
  - Проверка, есть ли у пользователя почтовый ящик
  - Проверка, добавлен ли пользователь в группы
- При провале любой из проверок дополняется глобальная переменная сбора ошибок соответсвующим сообщением.
- Если переменная сбора ошибок во всём скрипте включает в себя любые значения, то формируется скрытое сообщение на портал с указанием ошибок
- Если переменная сбора ошибок пустая, то
  - Формируется видимое сообщение на портал - таблица с указанием логина, пароля и почтового ящика сотрудника
  - Формируется скрытое сообщение на портал с указанием всех выданных прав доступа
- Через API BITRIX назначается ответсвенный по заявке с использованием графика дежурств
- Через API BITRIX отправляются ранее сформированные сообщения
  - При наличии ошибок отпрвляется скрытый комментарий к заявке с описанием собранных ошибок
  - При осутсвии ошибок отправляется сообщение с информацией по пользователю открытым комментарием и списком прав скрытым
- При отсутсвии ошибок переводим заявку на тестирование


### Функция Rename-Files
Функция переименовывания файлов выгрузки. 

Получает на вход идентификационное сообщение указывающее, какие именно работы проводились с пользователем: 
_UserUpdated - обновлены атрибуты пользователя 
_UserRestored - пользователь восстановлен из отключенных 
_UserCreated - создан новый пользователь
_UserDisabled - учётная запись пользователя отключена
Результат работы функции - переименованные файлы в папках выгрузки.

#### Логика работы
 - Проверка получения входных данных - идентификационного сообщения. При наличии:
   - Формирование нового имени для файлов
   - Проверка, к какому типу относится выгрузка - есть ли .csv, есть ли .jpg
   - Переименовывание всех файлов, найденных по выгрузке


### Функция Create-User-AD
Функция создания учётной записи AD.
При создании заполняет всевозможные атрибуты, представленные в выгрузке. Для выгрузок csv+фото устанавливает фотографию пользователю.

Результат работы функции - новая учётная запись в AD.

#### Логика работы
- Формирование пути домашней папки на основе логина
- Создание нового пользователя с параметрами:
  - Сервер - Server
  - Полное имя - Name
  - Имя Отчество - GivenName
  - Фамилия - Surname
  - Логин - SamAccountName
  - Имя для входа - UserPrincipalName
  - Отображаемое имя - DisplayName
  - Должность пользователя - Title
  - Описание ползьователя - Description
  - Отдел - Department
  - Домашняя папка - HomeDirectory
  - Домашний диск - HomeDrive
  - Юридическое лицо - Company
  - Мобильный телефон - MobilePhone
  - Рабочий телефон - OfficePhone
  - Требование сменить пароль при следующем входе - ChangePasswordAtLogon
  - Стандартная OU - Path
  - Стандартный пароль - AccountPassword
  - Включен ли аккаунт - Enabled
  - Расширенный атрибут с датой рождения - extensionAttribute5
  - Расширенный атрибут с ID сотрудника в 1С - extensionAttribute6
  - Расширенный атрибут с именем пользователя - extensionAttribute7
  - Расширенный атрибут с ID должности сотрудника в 1С - extensionAttribute10
  - Расширенный атрибут с ID отдела сотрудника в 1С - extensionAttribute11
- Проверка, есть ли фото по данной выгрузке. Если есть:
  - Получаем наименование файла с расширением и передаём его в Python скрипт по обнаружению лица и обрезанию фото под размер AD
  - Меняем фото в учётной записи AD сотрудника


### Функция Test-Profile-Folder
Функция создания/восстановления домашней директории пользователя. Производит поиск папки пользователя в директориях активных и уволенных пользователей. 

Результатом работы функции является домашняя папка пользователя в директории активных пользователей, а также полностью выданные пользователю права на его домашнюю папку.

#### Логика работы
- Получаем пути домашней папки в директориях активных и уволенных пользователей по логину пользователя
- Проверяем, существует ли домашняя папка в директории для активных пользователей. Если не существует:
  - Проверяем, существует ли домашняя папка в директории для уволенных пользователей. Если не существует:
    - Создаём новую папку в директории активных пользователей
  - Если папка пользователя существует в директории уволенных:
    - Перемещаем её в директорию активных
    - Выдаём права учётной записи пользователя на свою домашнюю папку


### Функция Update-User-AD-info
Функция обновления атрибутов пользователя AD. Сверяет полученную из выгрузки информации с тем, что есть в AD. 
Составляет два словаря - на обновление и очищение атрибутов. Производит обновление и очищение атрибутов согласно составленным словарям.

Результатом работы являются изменённые атрибуты пользователя согласно выгрузке.

#### Логика работы
- Очищение хеш-таблицы с заполненными в выгрузке значениями и массива с пустыми в выгрузке значениями. По результату дальнейших проверок данные переменные будут заполняться соответвенно. 
- Проверка полного соответсвия ФИО. По результатм проверки:
  - Переименовывание объекта AD - учётной записи в AD сотрудника.
  - При отличиях в AD и выгрузке смена параметра Surname - данный параметр не изменяется через хеш-таблицу
  - При отличиях в AD и выгрузке добавление в хеш-таблицу таких параметров, как:
    1. DisplayName
    2. GivenName
    3. extensionAttribute7
- Проверка соответсвия наименования должности и ID должности в 1С. По результатм проверки, при нахождении отличий в AD и 1С - добавление в хеш-таблицу таких параметров, как:
  1. Title
  2. Description
  3. extensionAttribute10
- Проверка соответсвия наименования отдела и ID должности в 1С. По результатм проверки, при нахождении отличий в AD и 1С - добавление в хеш-таблицу таких параметров, как:
  1. Department
  2. extensionAttribute11
- Проверка соответсвия Юр.лица. По результатм проверки
  1. Если Юр. лицо заполнено и отличается от заполненного в AD - добавление параметра Company в хеш-таблицу
  2. Если Юр. лицо не заполнено в выгрузке - добавление параметра Company в массив на очищение
- Проверка соответсвия мобильного телефона. По результатм проверки точно такая же работа производится с параметром Mobile
- Проверка соответсвия даты рождения. По результатм проверки работа производится с параметром extensionAttribute5
- Проверка соответсвия адреса работы. По результатм проверки работа производится с параметром StreetAddress
- Проверка соответсвия руководителя. По результатм проверки работа производится с параметром Manager
- Проверка соответсвия имени для входа. При отличиях параметра в AD от шаблона "AD_login@maximum-avto.ru" параметр UserPrincipalName добавляется в хеш-таблицу
- Проверка типа выгрузки - если выгрузка включает в себя фото, то
  1. Получаем наименование файла с расширением и передаём его в Python скрипт по обнаружению лица и обрезанию фото под размер AD
  2. Записываем параметр thumbnailPhoto в хеш-таблицу со значением равным пути к обработанному фото
- Очищаем все атрибуты, записанные в массиве на очищение
- Заменяем все атрибуты, записанные в хеш-табилце на замену


### Функция OU-User-Change 
Функция смены OU учётной записи пользователя.
Совершает поиск OU, в которой находятся пользователи идентичные пользователю в выгрузке по порядку: 
 1. Идентичные отдел и должность
 2. Идетичный отдел
 3. Идентичный руководитель и должность

Если совпадения найдены, то OU первого найденного пользователя приравнивается к OU для переноса пользователя в выгрузке.
Если не найдено совпадений среди всех включенных пользователей, не находящихся в ou disabled users, по всем поискам, то перенос совершается в стандартную OU.

Результатом работы функции является перенесённая в корректную OU учётная запись пользователя.

#### Логика работы
- Очищение переменных - счётчиков циклов и переменной - флага нахождения пользователя, с которого копируем OU
- Поиск сотрудников с идентичным отделом и должностью. Если такие есть:
  - Проверяем, что сотрудник не в OU disabled_users. Если OU найденного сотрудника удовлетворяет этим требованиям:
    - #### Блок окончания поиска OU:
      - Фиксируем OU сотрудника и его логин
      - Назначаем переменную - флаг нахождения пользователя, с которого копируем -  в $true
  - Если сотрудник в OU disabled переходим к следующему, пока не найдём нужного. Если нашли:
      - [Блок окончания поиска OU](#блок-окончания-поиска-OU)
- Если ни один пользователь из найденных не подошёл под критерии или же соответсвий по первому поиску не было вовсе:
    - Переходим ко второй проверке - ищем сотрудников из того же отдела, что и в выгрузке, которые находятся не в OU disabled. Если такие есть:
      - [Блок окончания поиска OU](#блок-окончания-поиска-OU)
- Если ни один пользователь из найденных не подошёл под критерии или же соответсвий по второму поиску не было вовсе:
    - Переходим к третьей проверке - сначала проверяем, что в выгрузке указан руководитель, затем ищем сотрудников с таким же руководителем и должностью, которые находятся не в OU disabled. Если такие есть:
      - [Блок окончания поиска OU](#блок-окончания-поиска-OU)
 - Если по результатам трёх проверок OU для пользователя не нашлась, то устанавливаем OU для переноса - стандартную OU.
 - Проверяем, есть ли отличия между найденной OU и OU, в которой сейчас находится пользователь
   - Если есть, делаем запись о переносе в лог файл и перемещаем учётную запись в новую OU
   - Если отличий между текущей и найденной для переноса OU нет, записываем в лог-файл информацию о том, что перенос не требуется


### Функция Check-Zoom
Функция проверки Zoom аккаунта пользователя. Использует библиотеку PSZoom 2.0.4.1
Совершает подключение к API Zoom, поиск учётной записи сотрудника, обновление атрибутов учётной записи при расхождениях. При отсутсвии учётной записи создаёт новую.

Результатом работы функции является Zoom аккаунт пользователя с актуальной в нём информацией

#### Логика работы
- Подключение к Zoom с помощью бибилотеки PSZoom
- Получение списка всех пользователей. Если удалось получить список:
  - Поиск аккаунта Zoom сотрудника из выгрузки
  - Проверка, нашли ли аккаунт. При нахождении аккаунта:
    - Получение всех расширенных атрибутов Zoom пользователя
    - Проверка, совпадают ли данные в аккаунте Zoom с выгрузкой в параметрах:
      1. Имя
      2. Фамилия
      3. Отдел
      4. Должность
    - При расхождениях в значяениях - обновление значений соответствующих атрибутов в аккаунте Zoom
    - При отсутсвии расхождений - запись в лог файл информации о том , что изменения не требуются 
  - При отсутсвии аккаунта среди всех пользователей Zoom
    - Создание нового пользователя Zoom
    - Заполнение расширенных атрибутов
- Если список всех пользователей Zoom получить не удалось - дополняется глобальная переменная сбора ошибок соответсвующим сообщением.

### Функция Create-Update-Mailbox
Функция проверки почтового ящика пользователя. Эта функция проверяет наличие ящика на почтовом сервере по логину, проверяет в какой базе находится ящик, если он существует и перемещает его, если текущая база является disabled. При отсутсвии ящика создаёт его для пользователя. 

Результат работы функции - активный почтовый ящик пользователя в рабочей активной базе почтовых ящиков с включённым EWS.

#### Логика работы
- Подключение к почтовому серверу
- Поиск почтового ящика пользователя по логину. Если почтовый ящик найден 
  - Удаляем старые задания на перемещение почтового ящика
  - Проверяем, не находится ли ящик в disabled. Если находится
    - Запускаем миграцию почтового ящика в базу активных почтовых ящиков
- Если почтовый ящик не найден - создаём новый
- Выполняем проверку, что почтовый ящик существует. Если существует 
  - Выполняем проверку установленного параметра EWS. Если выключен
    - Устанавливаем значение EWS в стандартное значение $null
  - Если EWS включен, записываем в лог информацию о том, что EWS работает
- Если проверка создания почтового ящика не удачна, то дополняется глобальная переменная сбора ошибок соответсвующим сообщением.
- Отключаемся от почтового сервера


### Функция Remove-Add-user-from-AD-groups
Функция обновления прав пользователя в AD. Очищает все прошлые группы, кроме стандартных групп и уникальных групп рассылки на пользователя. Добавляет группы по умолчанию, группы-шаблоны должности и отдела

Результат работы функции - корректно настроенные права на учётной записи пользователя согласно его должности и отделу

#### Логика работы
- Очищение глобальных переменных групп-шаблонов должности и отдела
- Очищение старых групп пользователя. Удаляются все группы, кроме:
  1. Domain Users
  2. Группы рассылки с именем группы = ФИО пользователя и фамилией пользователя
- Добавление пользователя в стандартные группы для всех пользователей
- Если в выгрузке указан ID должности
  - Поиск шаблона должности в AD по ID должности в 1С. Переопределние глобальной переменной группы-шаблона должности. Если группа-шаблон по данной должности существует
    - Добавление пользователя в эту группу
  - Если шаблона долности в AD не существует - запись в лог информации об этом
- Если в выгрузке не указан ID должности, то дополняется глобальная переменная сбора ошибок соответсвующим сообщением
- Если в выгрузке указан ID отдела
  - Поиск шаблона отдела в AD по ID отдела в 1С. Переопределние глобальной переменной группы-шаблона отдела. Если группа-шаблон по данному отделу существует
    - Добавление пользователя в эту группу
  - Если шаблона отдела в AD не существует - запись в лог информации об этом
- Если в выгрузке не указан ID отдела, то дополняется глобальная переменная сбора ошибок соответсвующим сообщением
- Если по результатам отработки функции глобальная переменная хотя бы одной из групп-шаблонов осталась не определённой
  - [Создание нового шаблона прав в AD - Create-AD-Template](#функция-Create-AD-Template)


### Функция Create-AD-Template
Функция создания шаблона должности или отдела. Создаёт группу в AD - шаблон должности или отдела: сначала формирует имя для группы, создаёт универсальную группу безопасности, почту для группы и заполняет атрибуты необходимые расширенные атрибуты.
Далее создаёт заявку на портал в ТП ИТ для заполнения нового шаблона прав: формирует заголовок, текст заявки и создаёт её в указанную категорию на портале.
id категории, куда на текущий момент распределяется заявка: 38 - ИТ Руставели
Возможные для распрделения категории: 
- 39 - ИТ Форд/Хонда
- 37 - ИТ Торфяная
- 76 - ИТ Непокоренных
- 119 - ИТ Рыбинская
- 120 - ИТ Транспортная
- 124 - ИТ Удалённая работа
- 105 - ИТ 2 линия
- 106 - ИТ 3 линия

Результат отработки функции - универсальные группы безопасности по новому шаблону со стандартными правами, запущенный процесс ручного наполнения новых групп правами.

#### Логика работы
- Проверка, определена ли глобальная переменная группы-шаблона долности. Если не определена:
  - ####Блок создания новой группы-шаблона прав в AD
    - Получаем полное наименование группы из выгрузки
    - Обрезаем длину наименования до 48-ми символов
    - Составляем наименование по шаблону
      - Access_ + обрезанное до 48-ми символов наименование + первые значения ID группы в 1с
    - Проверяем, что составленное имя подходит под формат AD. Обрезаем до 64-ёх символов, если это не так
    - Составялем алиас для будущей почты. Создаём его по составленному имени группы [функцией транслитерации - Translit](#функция-Translit)
    - Составляем SamAccountName, меняя тире на нижнее подчёркивание в ID группы в 1С
    - Создаём новую группу безопасности в указанной OU
    - Заполняем extensionAttribute6 у группы = ID группы в 1С
    - Если группа создалась
      - Подключение к почтовому серверу
      - Создание ящика для группы безопасности
      - Убираем ящик из зоны видимости в списке адресатов
      - Отключаемся от почтового сервера
      - Добавляем стандартные права для группы-шаблона
      - Добавляем сотрудника из выгрузки в новую группу-шаблон
      - Формируем заголовок и текст заявки на портал
      - Создаём заявку на портале на заполнение нового шаблона прав через API Bitrix, указывая категорию для создания заявки
      - Получаем номер созданной заявки
      - Если код ответа от портала = 200, то записываем в логирование информацию, что заявка создана и указываем номер заявки
      - Если код ответа отличается - дополняется глобальная переменная сбора ошибок соответсвующим сообщением
    - Если группа не создалась - дополняется глобальная переменная сбора ошибок соответсвующим сообщением
- Проверка, определена ли глобальная переменная группы-шаблона отдела. Если не определена:
  - [Блок создания новой группы-шаблона прав в AD](#блок-создания-новой-группы-шаблона-прав-в-AD)
 
 ===================================================================
