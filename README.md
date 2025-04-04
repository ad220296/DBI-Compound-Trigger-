# DBI-Cpmpound-Trigger

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
