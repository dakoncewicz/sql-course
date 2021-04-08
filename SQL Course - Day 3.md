# SQL Course - Day 3
Plan :
1. Procedury składowane
2. Wyzwalacze i eventy
3. Transakcje i współbieznośc
4. Typy danych w SQL
5. Indeksowanie tabel w bazie danych
6. Bezpieczeństwo baz danych

## Section 1 : Procedury składowane             (60 minut)
### Lesson 1 : Wstęp - co to są procedury składowane ?
- Obiekt przechowywany w bazie danych
- Kod SQL przechowywany po stronie bazy danych, który może zostać wykonany wielokrotnie. Można wprowadzić do nich instrukcje sterujące znane z języków programowania oraz (prawie) wszystkie polecenia, operatory SQL:
  
    CREATE PROCEDURE GetProductsDetails(IN cpu VARCHAR(20) )
    BEGIN
        SELECT * 
        FROM Komputer INNER JOIN Producent 
            ON Komputer.Id = Producent.ID 
        WHERE Procesor = cpu;
    END;    


#### Procedury składowane vs Kod wbudowany (jdbc, orm)
- umozliwiają przechowywaie i organizowanie kodu po stronie serwera
  - wprowadzają poziom abstrakcji - pomagają ukryc złozonoś struktury bazy danych
  - rozdzielenie odpowiedzialności = java + baza danych
- szybsze wykonanie (lepszy performance)
- bezpieczeństwo danych

### 2 : Tworzenie procedur składowanych
#### Zaczynamy :
- mamy proste zapytanie SQL
    SELECT * FROM clients

- stwórzmy z niego procedurę składowaną :
    DELIMITER $$  -- specyficzne dla MySQL
    
    CREATE PROCEDURE get_clients( )
    BEGIN
        SELECT * FROM clients;
    END$$       -- kończymy nowym Delimiterem

    DELIMITER ; --powracamy do konwencji

- wywołujemy procedurę
  - z lewego panelu w MySQLWorkbench
  - pisac wywołanie :  CALL get_clients();

#### Zadanie 1  ----------------------------------------------------
Utwórz procedurę składowaną get_invoices_with_balance(), zwracającą wszystkie faktury dla których balance > 0

Rozwiązanie :
- najpierw zapytanie, później obudujemy je procedurą

DELIMITER $$
CREATE PROCEDURE get_invoices_with_balance()
BEGIN
    SELECT *
    FROM invoices_wirh_balance
    WHERE balance > 0
END$$

DELIMITER ;

### 3 : Generowanie procedur składowanych z uzyciem MySQLWorkbench
- ustawiamy się w bazie danych na katalogu Stored Procedure
- right-click > z menu kontekstowego wybieramy Create Store Procedure.
- generuje się szkielet procedure w który wpisujemy kod
- po zakończeniu uruchamiamy przycisk [Apply and Revert]
- Pojawia się ekran z wygenerowanym kodem (w tym polecenie USE, DELIMITER etc.)
- Akceptujemy wygenerowany kod [Apply] - KONIEC

### 4 : Usuwanie procedur składowanych
- Składnia :
    DROP PROCEDURE 'name';

- good practice aby uniknąc błędu wykonania w przypadku gdy procedura nie isynieje 
    DROP PROCEDURE IF EXISTS get_clients;

- polecenie kasuje procedurę z bazy danych - aby zachowac jej kod nalezy uzyc np. odrębnego katalogu i systemu kontroli wersji.


### 5 : Parametry 
#### Wstęp
    DROP PROCEDURE IF EXISTS get_clients_by_state;

    DELIMITER $$
    CREATE PROCEDURE get_clients_by_state(
        state CHAR(2)       -- VARCHAR
    )
    BEGIN
        SELECT * FROM clients c
        WHERE c.state = state;
    END$$

    DELIMITER ;

#### Zadanie 2  ----------------------------------------------------
Napisz w bazie danych sql_invoicing procedurę składowaną zwracającą faktury dla klienta o podanym id. Uzyj generatora kodu MySQL
get_invoices_by_client( IN client_id)

#### Parametry z wartością domyślną
- nie ma tutaj wielkiej pomocy ze strony SQL, trzeba to obsłuzyc w kodzie - przypadek 1
  - jeśle wartośc parametru jest NULL zwróc klientów dla California
    // ..
    BEGIN
        IF state IS NULL THEN
            SET state = 'CA'
        END IF
        SELECT *  -- reszta jak poprzednio
        // ...
    END

- Przypadek 2 
  - jeśli wartośc parametru jest NULL zwróc wszystko
    // ...
    SELECT *
    FROM client c
    WHERE  c.state = IFNULL(state, c.state)  -- IF NULL > c.state = c.state

### 6 : Walidacja parametrów
- w procedurach mogą byc uzyte inne konstrukcje, niz zapytania. Dostępne są wszystkie polecenia DML : INSERT, UPDATE, DELETE.
- Dostęptna jest większośc poleceń DDL np. 
  - CREATE TEMPORARY TABLE ...
  - DROP TABLE
  - etc

CREATE PROCEDURE make_payment (
    invoice_id INT,
    payment_amount DECIMAL(9, 2),
    payment_date DATE
)
BEGIN
    UPDATE invoices i
    SET
        i.payment_total  = payment_amout,
        i.payment_date = payment_date
    WHERE i.invoice_id = invoice_id;
END

- ale w przypadku błędnych parametrów zaburzymy dane w bazie danych - potrzebna jest kontrola parametrów wejściowych
  - kod defensywny
  - Przykład :

        //...
        IF payment_amount <= 0 THEN
            SIGNAL SQLSTATE '22003'     -- działa jak Throw Exception w java
                SET MESSAGE_TEXT = 'Invalid payment amount'
        END IF

- kod defensywny jednocześnie mocno komplikuje procedurę i odwraca uwagę od głównego bloku realizującego funkcjonalnoś procedury. A zatem :
  - ograniczaj kod defensywny do rozsądnego minimum
  - te sprawdzenia które mozna (np UNIQE, NOT NULL) pozostaw w gestii silnika bazy danych


### 7 Parametry wyjściowe

    CREATE PROCEDURE get_unpaid_invoices_for_client (
        client_id INT,
        OUT invoices_count INT,
        OUT invoices_total DECIMAL(9, 2)
    )
    BEGIN
        SELECT COUNT(*), SUM(invoice_total)
        INTO invoices_count, invoices_total
        FROM invoices i
        WHERE i.client_id = client_id 
            AND mayment_total = 0;
    END

- tak naprawdę zmienne OUT są zmiennymi sesyjnymi deklarowanymi niejawnie przed przekazaniem ich do procedury przez MySQLWorkbench

### 8 Deklarowanie i uzycie zmiennych
- Zmienne sesyjne lub zmienne uzytkownika - znikają po wylogowaniu się danego usera
    SET @invoices_count = 0

- Zmienne lokalne - zyją tak długo jak procedura w której są zadeklarowane.

CREATE PROCEDURE get_risk_factor()
BEGIN
    -- zmienne lokalne muszą by deklarowane na początku bloku 
    DECLARE risk_factor DECIMAL(9, 2) DEFAULT 0; 
    DECLARE invoices_total DECIMAL(9, 2);
    DECLARE invoices_count INT;

    SELECT COUNT(*), SUM(invoice_total)
        INTO invoices_count, invoices_total
        FROM invoices;

    -- risk_factor = invoices_total / invoices_count * 
    SET risk_factor = invoices_total / invoices_count * 5;

    SELECT risk_factor;
END

### 9 : Funkcje 
- to 'nasze' funkcje
- Są podobne do procedur, różnica polega m.in. na tym, że funkcje wykonywane zwracają rezultat, pojedyńczą wartośc; a procedury nie są do tego zobligowane.
- Powrócmy do przykładu z poprzedniej lekcji ale teraz wygenerujemy funkcję : katalog Functions > right+click > Create Function

CREATE FUNCTION get_risk_factor_for_client(
    client_id INT
)
    RETURNS INTEGER
    READS SQL DATA   -- inne : MODIFIES SQL DATA | DETERMINISTIC
BEGIN
    DECLARE risk_factor DECIMAL(9, 2) DEFAULT 0; 
    DECLARE invoices_total DECIMAL(9, 2);
    DECLARE invoices_count INT;

    SELECT COUNT(*), SUM(invoice_total)
    INTO invoices_count, invoices_total
    FROM invoices I
    WHERE I.CLIENT_ID = CLIENT_ID

    -- risk_factor = invoices_total / invoices_count * 
    SET risk_factor = invoices_total / invoices_count * 5;

    RETURN IFNULL( risk_factor, 0); -- ostatnia instrukcja w funkcji
END

- Uzycie jest identyczne jak w przypadku funkcji wbudowanych
- Przykład :
    SELECT 
        client_id,
        name,
        get_risk_factor_for_client(client_id) AS risk_factor
    FROM clients


## Section 2 : Wyzwalacze i eventy          (30 minut)
### Lesson 1 : Triggers (wyzwalacze)
- Tzw. triggery są to specjalne rodzaje procedur dołączane do tabel, które wykonują się automatycznie PRZED lub PO wykonywaniu operacji: INSERT, UPDATE, DELETE na rekordach  struktury, na której były wykonane ww operacje.
  
#### Tworzenie triggerów
- konwencja nazewnicza : table_after_insert | table_before_update etc.
- są dostępne dwa predefiniowane obiekty :
  - NEW - nowo utworzony obiekt np. po INSERCIE, po UPDATE
  - OLD - poprzedni obiekt np przed DELETE lub przed UPDATE

- przykład : automatyczny Update tablicy invoices[] przy rejestracji płatności (w tablicy payments[])

    DELIMITER $$

    CREATE TRIGGER payments_after_insert
        AFTER INSERT ON payments
        FOR EACH ROW
    BEGIN
        -- any SQL code 
        UPDATE invoices
        SET payment_total = payment_total + NEW.amount
        WHERE invoice_id = NEW.invoice_id;
    END $$

    DELIMITER ;

- testujemy nasz trigger
    INSERT INTO payments
    VALUES (DEFAULT, 5, 3, '2021-01-01', 10, 1) 

#### Zadanie 1 ----------------------------------------------------
Utwórz kolejny TRIGGER, który wykona operację przeciwną do wykonywanej przez payments_after_insert(), tzn po usunięcie płatności ZMNIEJSZA saldo (invoices.payment_total) na fakturze.

Rozwiązanie : 
    CREATE TRIGGER payments_after_delete
        AFTER DELETE ON payments
        FOR EACH ROW
    // ...
    SET payment_total = payment.total - OLD.amount
    WHERE invoice_id = OLD.invoice_id;
    // ... 

#### Przeglądanie (viewing) triggerów
- czy MySQLWorkbench ma przeglądanie triggerów? Sprawdzic !

    SHOW TRIGGERS

- lub 
    SHOW TRIGGERS LIKE 'payments%'  -- wszystkie triggery na tablicy 'payments'

#### Usuwanie triggerów
    DROP TRIGGER IF EXISTS payments_after_insert;

- good practicies : w skrypcie wszystkie CREATE : tablic, procedur, trigerów powinny byc poprzedzone operacją DROP - dla bezpieczeństwa, gwarancji utworzenia nowego obiektu w bazie

### 2 : Przykład - uzycie triggerów w celach audytowych
- Krok 1 : Utworzyc tablicę audytową
    USE sql_invoicing;
    DROP TABLE IF EXISTS payments_audit;

    CREATE TABLE payments_audit (
        client_id       INT
        date            DATE
        amount          DECIMAL(9, 2)
        action_type     VARCHAR(50)
        action_date     DATETIME
    )
- kod jest dostępny w skrypcie create-payments-table.sql

- Krok 2 : Utworzenie automatycznego triggera logującego zmiany w bazie audytowej
    - do triggera `payments_after_insert` dodajemy kod :
        //...
        INSERT INTO payments_audit
        VALUES (NEW.client_id, NEW.date, NEW.amount, 'Insert', NOW());
    END $$

- Krok 3 : sprawdzamy działanie zmodyfikowanego triggera
    INSERT INTO payments
        VALUES(DEFAULT, 5, 3, '2021-01-01', 10, 1 )

#### Zadanie 2 ----------------------------------------------------
Do triggera z poprzedniego cwiczenia, który po usunięcie płatności ZMNIEJSZA saldo (invoices.payment_total) na fakturze, dodaj kod zapisujący w logu audytu dokonaną zmmianę.

Rozwiązanie :
        //...
        INSERT INTO payments_audit
        VALUES (OLD.client_id, OLD.date, OLD.amount, 'Delete', NOW());
    END $$

### 3 : Zdarzenia (eventy)
- Wstęp
    - Zdarzenia MySQL to zadania wykonywane zgodnie z określonym harmonogramem (scheduled events). 
    - Zdarzenia MySQL to nazwane obiekty, które zawierają jedną lub więcej instrukcji SQL. Są przechowywane w bazie danych i wykonywane w jednym lub kilku odstępach czasu.
        - Na przykład : można utworzyć zdarzenie optymalizujące wszystkie tabele w bazie danych, które jest uruchamiane o godzinie 1:00 w każdą niedzielę.

    - Zdarzenia MySQL są również znane jako „wyzwalacze czasowe (temporal triggers)”, ponieważ są wyzwalane przez czas, a nie przez zdarzenia DML, tak jak zwykłe wyzwalacze. Zdarzenia MySQL są podobne do cronjob w systemie Linux lub harmonogramu zadań w systemie Windows.
  
#### Tworzenie eventów
- najpierw włączamy scheduler eventów w MySQL
- 
    SHOW VARIABLES LIKE 'event%'; -- sprawdzamy stan obecny
    SET GLOBAL event_scheduler = ON;

- tworzymy event

    DELIMITER $$
    DROP EVENT IF EXISTS yearly_delete_stale_audit_rows$$
    -- konwencja : frequency_operationDML_action

    CREATE EVENT yearly_delete_stale_audit_rows
        ON SCHEDULE
        -- AT '2020-01-01'
        -- EVERY 1 YEAR STARTS '2020-01-01' ENDS '2025-01-01'
    DO BEGIN
        DELETE FROM payments_audit
        WHERE action_date < NOW() - INTERVAL 1 YEAR;

        -- alternatywnie DATESUB( NOW(, INTERVAL 1 YEAR))
    END $$

    DELIMITER ;

#### Zmiana i usuwanie eventów
    SHOW EVENTS;
    SHOW EVENTS LIKE 'yearly%'

    DROP EVENT IF EXISTS yearly_delete_stale_audit_rows

- zmiana definicji/ deklaracji eventu
    zamiast CREATE EVENT  . ALTER EVENT
    // reszta kodu jak poprzednio

- włączenie / wyłączenie danego eventu
    ALTER EVENT yearly_delete_stale_audit_rows ENABLE;  -- lub DISABLE

## Section 3 : Transakcje i współbieznośc       (60 minut)
### Lesson 1 : Transakcje - wstęp
- Transakcje reprezentują zbiór zapytań na bazie danych, które reprezentują i tworzą logiczną całość. Powinny być one wykonane w całości bądź wcale (ang. single unit of work).
- Operacje te powinny być realizowane w ramach transakcji, w której można wyszczególnić 3 etapy:

1. rozpoczęcie transakcji                   START TRANSACTION
2. wykonanie instrukcji                     // Instrukcje SQL
3. zatwierdzenie lub wycofanie transakcji   COMMIT | ROLLBACK

### 2 : Tworzenie transakcji
- odtworzenie stanu początkowego bazy - reloading script
- przykład :

    USE sql_store 
    START TRANSACTION;

    -- pierwsza instrukcja
    INSERT INTO orders(customer_id, order_date, status)
    VALUES (1, '2021-01-01', 1);

    -- druga instrukcja
    INSERT INTO order_items
    VALUES (LAST_INSERTED_ID(), 1, 1, 1);

    COMMIT; -- zatwierdzenie obu instrukcji

- symulacja niewykonania transakcji
    - Querry Menu > Execute Current Statement (Cmd + Enter)
    - wykonujemy tylko pierwszą instrukcję (bez drugiej)
    - rozłączamy się z serwerem - zamykamy okno robocze MySQLWorkbench
    - wznawiamy połączenie w głównym oknie  MySQLWorkbench
    - transakcja nie została wykonana - został wykonany automatyczny ROLLBACK, po symulowanym rozłączeniu stacji roboczej z serwerem.

- autocommit - objęcie transakcją pojedynczej instrukcji, tak aby zawsze wykonała się w całości albo wcale.

    SHOW VARIABLE LIKE 'autocommit%'

### 3 : Współbieznośc i blokady (locking)

    USE sql_store;
    START TRANSACTION;

    UPDATE customers
    SET points = points + 10
    WHERE customer_id = 1;
    COMMIT;

    - Demonstracja Lock : dwa takie same skrypty uruchomione pod kontrolą "debugera" - lock - drugi wątek czeka na zakończenie pierwszego
      - albo timeout albo .... 

### 4 : Problemy ze współbieznością
Prezentacja : 

### 5 : Transaction isolation levels (opcjonalnie)
- czy dalej ciągnąc temat ? NIE
- prezentacja

## Section 4 : Typy danych - podsumowanie

## Section 5 : Indeksowanie tabel w bazie danych  (60 minut)
### Lesson 1 : Wstęp

### 2 : Tworzenie indeksów

### 3 : Przeglądanie indeksów

### 4 : Prefixowanie indeksów

### 5 : Indeksy pełnotekstowe (full-text index)

### 6 : Indeksy złozone (Composite indeks)

### 7 : Porządek kolumn w indeksie złozonym

### 8 : Kiedy indeksy są ignorowane przez silnik DB ?

### 9 : Uzycie indeksów do sortowania

### 10 : Covering index

### 11 : Zarządzanie indeksami (index maintenance)

## Section 6 : Bezpieczeństwo baz danych (30 minut)
### Lesson 1 : Tworzeie uzytkowników
#### Intro
- składnia : CREATE USER user_name@IP_address | ...@host_name | ...@domain
- przykład : jurek@127.0.0.1  lub adrian@sda.com lub adrian@'%.sda.com' -- dla domeny sda.com i wszystkich subdomen

- tworzymy uzytkownika :
  - CREATE USER darek IDENTIFIED BY '1234-567'; -- with no access restriction

#### Przeglądanie uzytkowników
- sposób 1) - bezpośrednio z tabel administracyjnych
    SELECT * FROM mysql.user;

- sposób 2) z panel administratora MySQLWorkbench
  - Menu > Administration > User and Privilages

#### Usuwanie uzytkoników
    DROP USER darek;

### 2 : Zmiana hasła uzytkownika
- z poziomu skryptu
    SET PASSWORD FOR darek = 'abcd-1234'

- z panelu nawigacyjnego MySQLWorkbench
    Menu > Administration > User and Privileges
    - zmień hasło
    - zatwierdź zmianę

### 3 : Nadawanie uprawnień (granting privileges)
#### Nadawanie uprawnień
- scenariusz 1 - web/desktop application
  - KROK 1
        CREATE USER moon_app IDENTIFIED BY 'ABCD-1234'

        GRANT SELECT, INSERT, UPDATE, DELETE, EXECUTE 
        ON sql_store.*  -- do wszystkich obiektów w bazie sql_store
        TO m

  - KROK 2 - utworzymy nowe połączenie > Home page
    - Setup new connection
    - testujemy nowe połączenie
  
- scenariusz 2 - admin
    GRANT ALL   -- wszystkie prawa
    ON *.*      -- do wszystkich baz i obiektów na serwerze
    TO darek;

#### Przegląd nadanych uprawnień
- z konsoli MySQLWorkbench
    SHOW GRANTS FOR darek; -- dla konkretnego usera

    SHOW GRANTS;    -- dla biezącego usera, teraz zalogowanego

- panelu administracyjnego (nawigacyjnego)
  - pokazac specyficzne uprawnienia dla moon_app
  
#### Revoking privileges
    GRANT CREATE VIEW
    ON sql_store.*
    TO moon_app;

    REVOKE CREATE VIEW
    ON sql_store.*
    FROM moon_app;