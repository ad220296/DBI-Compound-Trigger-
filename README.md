---
## 🔗 Navigation

- 🔄 [Zur Übersicht aller Trigger](https://github.com/ad220296/DBI-Trigger-Overview)
- 🔵 [Zu den INSTEAD OF Triggern](https://github.com/ad220296/DBI-Instead-Trigger-)
---

# 📘 COMPOUND Trigger in PL/SQL

## 📌 Einleitung: Was ist ein COMPOUND Trigger?

Ein **COMPOUND TRIGGER** wird auf eine **Tabelle** definiert (nicht auf Views!) und vereint mehrere Ausführungspunkte (Triggering Points) in einem einzigen Trigger.

Er kombiniert:
- **Statement-Level** Trigger: für die gesamte SQL-Anweisung
- **Row-Level** Trigger: für jede betroffene Zeile

➡️ Damit können komplexe Regeln abgebildet werden, die mit normalen Triggern nicht lösbar wären.  
Ein typisches Problem:  
> „Ein Mitarbeiter darf nicht mehr verdienen als der Bestverdiener (z. B. KING)“

In einem normalen **Row-Level Trigger** ist kein direkter Zugriff auf die Tabelle erlaubt – das löst der Compound Trigger durch den `BEFORE STATEMENT`-Block.

## 🔧 Typische Anwendungsfälle

- Ein Mitarbeiter darf max. 20 % mehr verdienen als der Durchschnitt seiner Abteilung.
- Ein Mitarbeiter darf nicht mehr verdienen als KING.
- Wenn letzter Mitarbeiter in einer Abteilung gelöscht wird → Abteilung automatisch mitlöschen.
- Zwischenspeicherung oder Protokollierung von Änderungen über mehrere Zeilen hinweg.

---

## 🔄 Allgemeines Compound-Trigger-Beispiel (Schablone)

```sql
CREATE OR REPLACE TRIGGER trigger_name                             -- 🟡 Trigger wird erstellt oder ersetzt
FOR INSERT OR UPDATE OR DELETE ON base_table                       -- 🔁 gilt für alle DML-Operationen (Insert, Update, Delete)
COMPOUND TRIGGER                                                   -- 🧩 Compound Trigger: vereint mehrere Trigger-Abschnitte

    -- 🔸 Deklarationsteil für Variablen, die in mehreren Abschnitten sichtbar sein sollen
    shared_var1 table.column%TYPE;                                 -- 📦 z. B. maximale Werte, Listen etc.
    TYPE mytable_t IS TABLE OF NUMBER INDEX BY PLS_INTEGER;        -- 🧺 Beispiel für Sammlung (Nested Table)
    shared_list mytable_t;                                         -- 🧺 Liste, die über alle Zeilen hinweg befüllt werden kann

-- 🔷 BEFORE STATEMENT: wird **einmal vor** der gesamten DML-Operation ausgeführt
BEFORE STATEMENT IS
BEGIN
    -- 🔍 z. B. Vorberechnungen, Maximalwerte ermitteln, Listen initialisieren
    SELECT MAX(sal) INTO shared_var1 FROM base_table;
END BEFORE STATEMENT;

-- 🔷 BEFORE EACH ROW: wird **für jede einzelne Zeile** vor dem Ändern ausgeführt
BEFORE EACH ROW IS
BEGIN
    -- 🔐 Zugriff auf :NEW und :OLD wie bei normalen Row-Level-Triggern
    IF :NEW.sal > shared_var1 THEN
        :NEW.sal := shared_var1;                                   -- 🧾 Begrenze z. B. neuen Wert auf Maximum
    END IF;
END BEFORE EACH ROW;

-- 🔷 AFTER EACH ROW: wird **nachdem eine einzelne Zeile geändert wurde** ausgeführt
AFTER EACH ROW IS
BEGIN
    -- ✅ z. B. Logik zur Protokollierung einzelner Änderungen
    dbms_output.put_line('Zeile verarbeitet: ' || :NEW.empno);
END AFTER EACH ROW;

-- 🔷 AFTER STATEMENT: wird **einmal am Ende** der DML-Operation ausgeführt
AFTER STATEMENT IS
BEGIN
    -- 🧹 z. B. Cleanup, Sammeloperationen, Ausgabe
    dbms_output.put_line('DML-Operation abgeschlossen.');
END AFTER STATEMENT;

END;
/
```

## 🧩 Aufgabe 1.2.1: Compound Trigger – Gehalt darf maximal 20 % über dem Durchschnitt liegen

### 🎯 Ziel
Beim `INSERT` oder `UPDATE` auf die Tabelle `emp` soll sichergestellt werden, dass das Gehalt nicht mehr als **20 % über dem aktuellen Durchschnittsgehalt** aller Mitarbeiter liegt.

Ein **Compound Trigger** berechnet einmal **vor der gesamten Operation** den Durchschnittswert und prüft **für jede betroffene Zeile**, ob das neue Gehalt zulässig ist. Falls nicht, wird es automatisch angepasst.

---

### 🧩 Trigger-Definition

```sql
set serveroutput on;

create or replace trigger trg_limit_sal_new_emp
for insert or update of sal on emp
compound trigger

    -- 🔸 Gemeinsame Variablen, sichtbar in allen Abschnitten
    avg_sal number;                                        -- 📦 Durchschnittsgehalt über alle Mitarbeiter
    max_sal number;                                        -- 📦 Obergrenze = 120 % vom Durchschnitt

    -- 🔷 1. Wird EINMAL vor dem gesamten DML-Statement ausgeführt
    before statement is
    begin
        select avg(sal) into avg_sal from emp;             -- 🔍 Durchschnitt berechnen
        max_sal := avg_sal * 1.2;                          -- 🧮 Obergrenze = Durchschnitt + 20 %
        dbms_output.put_line('⏱️ Erlaubtes Maximum: ' || round(max_sal, 2)); -- 🖨️ Info ausgeben
    end before statement;

    -- 🔷 2. Wird für JEDE betroffene Zeile einzeln ausgeführt
    before each row is
    begin
        -- ❗ Prüfung: Ist das neue Gehalt zu hoch?
        if :new.sal > max_sal then
            -- 🖨️ Hinweis auf Kürzung
            dbms_output.put_line('⚠️ Gehalt reduziert von ' || :new.sal || ' auf ' || round(max_sal, 2));
            -- ✂️ Gehalt kürzen
            :new.sal := max_sal;
        end if;
    end before each row;

end;
/
```

---

### 🧪 Testfälle

```sql
-- ✅ Insert mit zulässigem Gehalt
insert into emp (empno, ename, sal, deptno)
values (8001, 'OkayTest', 2500, 30);

-- ❌ Insert mit zu hohem Gehalt → wird automatisch gekürzt
insert into emp (empno, ename, sal, deptno)
values (8002, 'ZuvielTest', 9999, 30);

-- 🔍 Überprüfung: Haben die Kürzungen funktioniert?
select empno, ename, sal from emp where empno in (8001, 8002);
```
## 🧩 Aufgabe 1.2.2: Gehalt darf max. 20 % über dem Durchschnitt der eigenen Abteilung liegen

### 🎯 Ziel

Beim Einfügen oder Aktualisieren von Mitarbeiterdaten soll das Gehalt nicht mehr als **20 % über dem Durchschnitt der jeweiligen Abteilung** liegen.

Da der Durchschnitt **je Abteilung** berechnet werden muss, verwenden wir eine **PL/SQL-Collection (Nested Table)**, um die Werte zwischen Statement- und Row-Teil zu übertragen.

### 🧩 Trigger-Code

```sql
create or replace trigger trg_limit_sal_by_avg_per_dept
for insert or update of sal on emp 
compound trigger

    -- 📦 Tabelle zur Speicherung der Grenzwerte je Abteilung (deptno -> max_gehalt)
    type dept_avg_tab is table of number index by pls_integer;
    deptno_avgsal dept_avg_tab;

    -- 🔷 Wird einmal vor dem gesamten Statement ausgeführt
    before statement is 
    begin
        for rec in (select deptno, avg(sal) as avg_sal from emp group by deptno) loop
            deptno_avgsal(rec.deptno) := rec.avg_sal * 1.2;   -- 🧮 Maximalwert = Durchschnitt + 20 %
        end loop;
        dbms_output.put_line('⏱️ Durchschnittswerte pro Abteilung berechnet.');
    end before statement;

    -- 🔷 Wird für jede betroffene Zeile vor dem Einfügen/Aktualisieren ausgeführt
    before each row is
    begin
        if :new.sal > deptno_avgsal(:new.deptno) then         -- ❗ Überprüfung auf Abteilungslimit
            dbms_output.put_line(
                '⚠️ Gehalt (' || :new.sal || ') über Abteilungslimit (' 
                || round(deptno_avgsal(:new.deptno), 2) || ') → wird reduziert.');
            :new.sal := deptno_avgsal(:new.deptno);           -- ✂️ Anpassung durchführen
        end if;
    end before each row;

end;
/
```

### 🧪 Testfälle

```sql
-- ✅ Gehalt zu hoch für Abteilung → wird gekürzt
insert into emp (empno, ename, job, sal, deptno)
values (9001, 'MEGA', 'CLERK', 9000, 10);

-- ✅ Gehalt im Rahmen → bleibt bestehen
insert into emp (empno, ename, job, sal, deptno)
values (9002, 'OKAY', 'CLERK', 2500, 10);

-- ✅ Update mit zu hohem Wert → wird wieder gekürzt
update emp set sal = 9999 where empno = 9002;

-- 🔍 Kontrolle
select empno, ename, sal, deptno from emp where empno in (9001, 9002);
```

## 🧩 Aufgabe 1.2.3: Letzten Mitarbeiter einer Abteilung NICHT löschen

### 🎯 Ziel

Beim Löschen eines Mitarbeiters soll sichergestellt werden, dass **mindestens ein Mitarbeiter in jeder Abteilung verbleibt**.  
Wenn versucht wird, den **letzten Mitarbeiter einer Abteilung zu löschen**, wird ein **Fehler ausgelöst** und das Löschen verhindert.

### 💡 Idee

Da man in einem `BEFORE EACH ROW` Trigger **nicht direkt** auf dieselbe Tabelle zugreifen darf, wird die Anzahl der Mitarbeiter **vorher** (im `BEFORE STATEMENT`) für jede Abteilung berechnet und **zwischengespeichert**.  
Im Zeilen-Teil (`BEFORE EACH ROW`) wird dann mit dieser Information geprüft, ob der Mitarbeiter der **letzte** ist.

---

### 🧩 Trigger-Code

```sql
create or replace trigger trg_no_delete_last_emp
for delete on emp
compound trigger

    -- 📦 Assoziatives Array: speichert MA-Anzahl pro Abteilung (Index: deptno)
    type dept_emp_count_tab is table of pls_integer index by pls_integer;
    emp_count_by_dept dept_emp_count_tab;

    -- 🔷 Wird EINMAL vor dem gesamten Löschvorgang ausgeführt
    before statement is
    begin
        for rec in (select deptno, count(*) as emp_count from emp group by deptno) loop
            emp_count_by_dept(rec.deptno) := rec.emp_count;  -- 🧮 Anzahl zwischenspeichern
        end loop;
        dbms_output.put_line('⏱️ Mitarbeiteranzahl pro Abteilung wurde geladen.');
    end before statement;

    -- 🔷 Wird FÜR JEDE zu löschende Zeile einzeln ausgeführt
    before each row is
    begin
        if emp_count_by_dept.exists(:old.deptno) and emp_count_by_dept(:old.deptno) = 1 then
            raise_application_error(-20003, '❌ Letzter Mitarbeiter der Abteilung – Löschen nicht erlaubt!');
        end if;
    end before each row;

end;
/
```

---

### 🧪 Testfälle

```sql
-- ✅ Gültiger Löschvorgang: In der Abteilung sind noch andere MA
delete from emp where empno = 7369;

-- ❌ Ungültiger Löschvorgang: letzter MA → Fehler
delete from emp where empno = 9999;  -- Angenommen, 9999 ist der einzige in seiner Abteilung

-- 🔍 Kontrolle: MA-Anzahl je Abteilung
select deptno, count(*) from emp group by deptno;
```

---

## 🧩 Aufgabe 1.2.4: Letzten MA löschen → Abteilung automatisch mitlöschen

### 🎯 Ziel

Wenn der letzte Mitarbeiter einer Abteilung gelöscht wird, soll zusätzlich auch **die zugehörige Abteilung entfernt** werden.  
Das ist besonders sinnvoll, um verwaiste Datensätze in der `DEPT`-Tabelle zu vermeiden.

---

### 🧩 Trigger-Code

```sql
create or replace trigger trg_delete_last_emp_and_dept
for delete on emp
compound trigger

    -- 📦 Tabelle zur Speicherung der MA-Anzahl pro Abteilung
    type dept_emp_count_tab is table of pls_integer index by pls_integer;
    emp_count_by_dept dept_emp_count_tab;

    -- 🔷 Vor dem Statement: MA-Anzahl je Abteilung berechnen
    before statement is
    begin
        for rec in (select deptno, count(*) as emp_count from emp group by deptno) loop
            emp_count_by_dept(rec.deptno) := rec.emp_count;
        end loop;
        dbms_output.put_line('📋 MA-Zahl je Abteilung vorgemerkt.');
    end before statement;

    -- 🔷 Nach jeder Zeile prüfen: Letzter MA? Dann Abteilung löschen
    after each row is
    begin
        if emp_count_by_dept.exists(:old.deptno) and emp_count_by_dept(:old.deptno) = 1 then
            delete from dept where deptno = :old.deptno;
            dbms_output.put_line('🏢 Letzte MA-Zeile gelöscht → Abteilung ' || :old.deptno || ' wurde entfernt.');
        end if;
    end after each row;

end;
/
```

---

### 🧪 Testfälle

```sql
-- ✅ Löschung eines Mitarbeiters → andere bleiben → Abteilung bleibt bestehen
delete from emp where empno = 7839;

-- ✅ Löschung des letzten MA → Abteilung wird automatisch mitgelöscht
delete from emp where empno = 9001;

-- 🔍 Kontrolle
select * from dept where deptno not in (select deptno from emp);
```
# 🧩 Zusammenfassung: Compound-Trigger – Überblick der Aufgaben 1.2.1 bis 1.2.4

---

## 📌 Worum geht’s?

In den Aufgaben 1.2.1 bis 1.2.4 lernen wir verschiedene **Einsatzmöglichkeiten für Compound-Trigger** kennen – immer dann, wenn **Vorab-Berechnungen auf Tabellenebene** nötig sind, die **nicht direkt im Row-Trigger möglich** wären.

---

## 📊 Aufgaben im Überblick

| Aufgabe     | Thema / Regel                                                          | Aktion bei Verletzung              | Triggerteile genutzt              |
|-------------|------------------------------------------------------------------------|------------------------------------|-----------------------------------|
| **1.2.1**   | Gehalt darf max. 20 % über dem Durchschnitt **aller Mitarbeiter** liegen | Gehalt wird automatisch gekürzt    | `BEFORE STATEMENT`, `BEFORE EACH ROW` |
| **1.2.2**   | Gehalt darf max. 20 % über dem Durchschnitt **der eigenen Abteilung** liegen | Gehalt wird automatisch gekürzt    | `BEFORE STATEMENT`, `BEFORE EACH ROW` |
| **1.2.3**   | **Letzter Mitarbeiter** einer Abteilung darf **nicht gelöscht werden**     | Löschung wird verhindert (Fehler)  | `BEFORE STATEMENT`, `BEFORE EACH ROW` |
| **1.2.4**   | Wird der **letzte Mitarbeiter** gelöscht, wird die **Abteilung mitgelöscht** | Abteilung wird gelöscht automatisch | `BEFORE STATEMENT`, `AFTER EACH ROW`  |

---

## 🆚 Was ist der Unterschied zwischen 1.2.3 und 1.2.4?

| Vergleichspunkt                 | **1.2.3 – Verhindern**                             | **1.2.4 – Mitlöschen**                                |
|--------------------------------|----------------------------------------------------|--------------------------------------------------------|
| ❗ Ziel                         | Letzten MA **nicht löschen lassen**               | Letzten MA löschen **und Abteilung mitlöschen**       |
| 🚫 Verhalten bei letzter MA     | `raise_application_error`                         | `delete from dept`                                     |
| 🧠 Trigger-Logik                | Vor dem Löschen prüfen                            | Nach dem Löschen prüfen                                |
| 🔁 Triggerabschnitt             | `BEFORE EACH ROW`                                 | `AFTER EACH ROW`                                       |
| ⏱️ Zeitpunkt                   | Prüfung passiert **vor der Zeile**                | Aktion passiert **nachdem die Zeile gelöscht wurde**   |

---

## 💡 Wann was verwenden?

| Situation                                                 | Lösung                      |
|------------------------------------------------------------|-----------------------------|
| Werte über alle Zeilen hinweg vergleichen                 | `BEFORE STATEMENT` nutzen   |
| Einzelne Zeilen kontrollieren oder anpassen               | `BEFORE EACH ROW` nutzen    |
| Reaktion **nachdem** etwas gelöscht oder geändert wurde   | `AFTER EACH ROW` nutzen     |
| Kombination aus beiden Ebenen nötig                       | Compound-Trigger verwenden  |

