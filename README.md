# my-dbo-lab
DBO
).

---


2. «КЕЙС-ВОПРОСЫ»

(Вопрос – Краткий алгоритм – Команды/скрипты – Фраза-«чек»)

Q1. «Клиент не может войти в ДБО, что делаете?»  
1) Уточнить канал: web или мобильное приложение.  
2) Проверить статус сервиса:

   Windows: `sc query "Optima24Web"` или `Get-Service OptimaWeb`

   Linux: `systemctl status optima-tomcat`  
3) Смотрим лог:

   Windows: `C:\inetpub\wwwroot\Optima24\Logs\error-YYYY-MM-DD.log`

   Linux: `/opt/tomcat/logs/catalina.out | grep -i "401\|403"`  
4) Если 401 – проверяем LDAP: `ldapsearch -x -LLL -h 10.0.0.5 -b "ou=users,dc=bank,dc=kg" "(uid=client_login)"`

Фраза-чек: «В 90 % случаев проблема в блокировке по кол-ву неудачных входов – сбрасываю счётчик failAttempts в OpenLDAP».

Q2. «После релиза пропали счета клиента».

Алгоритм:  
- Проверяем changelog: `SELECT * FROM release_patch WHERE patch_id = '24.08.06.01';`  
- Запрос к БД:  

```sql
SELECT a.account_id, a.status, p.patch_id
FROM accounts a
JOIN patch_history p ON a.updated_by_patch = p.id
WHERE a.client_id = :client AND p.patch_date >= CURRENT_DATE - 1;
```  

- Если status = ‘HIDDEN’ – откатываем флаг: `UPDATE accounts SET status='ACTIVE' WHERE …`.

Фраза-чек: «Проблема в скрипте 2024-08-06-01-accountVisibility.sql – добавил отсутствующее условие WHERE branch = 'ALL'».

Q3. «Ошибка платёжного шлюза: код 09».  
- Коды «Элкарт»: 09 = «Insufficient funds».  
- Проверяем баланс клиента:  

```sql
SELECT balance
FROM card_accounts
WHERE pan_mask = '5559********1234';
```  

- Если денег нет – пишем клиенту.  
- Если деньги есть – смотрим лог шлюза: `grep '55591234' /var/log/sanarip-tolov/gateway.log`.

Фраза-чек: «Код 09 – это ответ core-banking, а не наша ошибка, но я всё равно проверю очередь RabbitMQ на предмет задержки».

Q4. «Клиент просит поднять лимит 500 000 → 2 000 000 сом».  
- Шаблон заявки в Jira: `LIM-2024-08-006`.  
- SQL-шаблон для проверки текущих лимитов:  

```sql
SELECT channel_id, daily_limit, monthly_limit
FROM client_limits
WHERE client_id = 123456;
```  

- Обновление:  

```sql
UPDATE client_limits
SET daily_limit = 2000000, updated_by = 'tech_support_ismailov'
WHERE client_id = 123456 AND channel_id = 'WEB';
```  

- Фиксируем в аудите: `INSERT INTO limit_audit …`.

Фраза-чек: «Согласно Положению НБКР №44, лимит >1 млн требует двух подписей: я отправил запрос на подпись начальнику службы безопасности».

---

3. SQL-ЗАПРОСЫ «НА ЗАМЕТКУ»

(учи наизусть – именно такие спрашивают)

1. Список неуспешных транзакций за сегодня:  

```sql
SELECT id, external_id, status, error_code, amount, created_at
FROM payment_tx
WHERE DATE(created_at) = CURRENT_DATE
  AND status NOT IN ('COMPLETED', 'PENDING');
```

2. Поиск дублей по ИНН:  

```sql
SELECT inn, COUNT(*) cnt
FROM clients
GROUP BY inn
HAVING COUNT(*) > 1;
```

3. XML-фрагмент для добавления нового мерчанта в шлюз «Элкарт»:  

```xml
<merchant id="12345">
  <name>O!Store KG</name>
  <terminal id="12345001" type="WEB"/>
  <callback>https://pay.ostore.kg/callback</callback>
</merchant>
```

---

4. WINDOWS / LINUX КОМАНДЫ



Windows:
- `eventvwr.msc` → Windows Logs → Application → фильтр по Source = «Kompanion24».  
- `netstat -ano | findstr :443` – проверка занятости порта.  
- `iisreset /stop && iisreset /start` – быстрый рестарт IIS-сайта.

Linux:
- `journalctl -u tomcat -f` – живой лог Tomcat.  
- `tcpdump -i eth0 host 10.0.0.5 and port 443 -w capture.pcap` – снимаем трафик.  
- `openssl s_client -connect 10.0.0.5:443 -showcerts` – проверка TLS-сертификата ЦБ.

---

5. КРИПТОГРАФИЯ 

- В КР используется ГОСТ 34.10-2012 через библиотеку `CryptoPro CSP`.  
- Файл ключа – .jks для Java, .pfx для Windows.  
- Проверка ЭЦП в xml-документе:

`xmlsec1 --verify --pubkey-cert-pem cert.pem --id-attr:Id file.xml`

Фраза-чек: «Для интеграции с «Элкарт» мы подписываем запросы detached-подписью ГОСТ 34.10 и кладём <> в заголовок SOAP».

---




