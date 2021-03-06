# Databanken 2

In dit document nemen we samen een duik in het hoofd van Wim Bertels. 

Ik zal ook proberen zo veel mogelijk codevoorbeelden in het document te steken om te helpen bij het oplossen van de vragen.

<!--ts-->
   * [Databanken 2](#databanken-2)
   * [1. Introductie](#1-introductie)
   * [2. Indexen &amp; Optimalisaties](#2-indexen--optimalisaties)
      * [Indexen](#indexen)
         * [Wat is een index?](#wat-is-een-index)
         * [Soorten indexen](#soorten-indexen)
      * [Optimalisaties](#optimalisaties)
   * [3. Explain](#3-explain)
      * [Dedection and benchmarking](#dedection-and-benchmarking)
         * [Statistieken](#statistieken)
         * [Benchmarking](#benchmarking)
   * [4. Limiting result sets &amp; lateral](#4-limiting-result-sets--lateral)
      * [Limiting result sets](#limiting-result-sets)
      * [Lateral joins](#lateral-joins)
   * [5. Procedurele SQL](#5-procedurele-sql)
      * [PL/pgSQL](#plpgsql)
      * [Functions vs Procedures](#functions-vs-procedures)
      * [Functions](#functions)
         * [Select into](#select-into)
         * [Perform &amp; Found](#perform--found)
         * [Exceptions](#exceptions)
         * [Beveiliging](#beveiliging)
         * [Immutability](#immutability)
      * [Procedures](#procedures)
      * [Triggers](#triggers)
   * [6. GDPR](#6-gdpr)
      * [Hoofdpunten](#hoofdpunten)
      * [Database admins](#database-admins)
      * [Info dissemination](#info-dissemination)
      * [Application builders](#application-builders)
      * [Data breaches](#data-breaches)
   * [7. Vensterfuncties](#7-vensterfuncties)
   * [8. CTEs en Beveiliging](#8-ctes-en-beveiliging)
      * [CTEs](#ctes)
         * [Oneindige lussen](#oneindige-lussen)
      * [Beveiliging](#beveiliging-1)
   * [9. ODMS, DBMS en ORDBMS](#9-odms-dbms-en-ordbms)
      * [ORDBMS](#ordbms)
         * [RULEs](#rules)
      * [NoSQL](#nosql)
         * [ACID](#acid)
         * [CAP](#cap)
   * [10. XML en JSON](#10-xml-en-json)
      * [XML](#xml)
         * [Voor- en nadelen](#voor--en-nadelen)
      * [XML en postgres](#xml-en-postgres)
      * [JSON](#json)
         * [JSON en postgres](#json-en-postgres)
      * [Andere types](#andere-types)
   * [Replicatie](#replicatie)
         * [Hoe doe je dat?](#hoe-doe-je-dat)
      * [Buffers](#buffers)

<!-- Added by: martijn, at: Sat Jun 13 18:41:25 CEST 2020 -->

<!--te-->

# 1. Introductie

Een beetje herhaling van vorig jaar. Hoe maak je een databank, hoe schrijf je queries, hoe doe je een insert, ...

```sql
SELECT [ ALL | DISTINCT [ ON ( expression [, ...] ) ] ]
 [ * | expression [ [ AS ] output_name ] [, ...] ]
 [ FROM from_item [, ...] ]
 [ WHERE condition ]
 [ GROUP BY grouping_element [, ...] ]
 [ HAVING condition [, ...] ]
 [ WINDOW window_name AS ( window_definition ) [, ...] ]
 [ { UNION | INTERSECT | EXCEPT } [ ALL | DISTINCT ] select ]
 [ ORDER BY expression [ ASC | DESC | USING operator ] [ NULLS { FIRST |
 LAST } ] [, ...] ]
 [ OFFSET start [ ROW | ROWS ] ]
 [ FETCH { FIRST | NEXT } [ count ] { ROW | ROWS } ONLY ]
```

Grouping elements van **group by**

```sql
( )
expression
( expression [, ...] )
ROLLUP ( { expression | ( expression [, ...] ) } [, ...] )
CUBE ( { expression | ( expression [, ...] ) } [, ...] )
GROUPING SETS ( grouping_element [, ...] )
```

Similar to

```sql
'abc' SIMILAR TO '%(b|d)%' 
```

deze shit ook

> BETWEEN – WHERE x between 1 and 100 
>
> OVERLAPS – WHERE (datum1,datum2) OVERLAPS (datumA, datumB) 
>
> IS NULL – WHERE x IS NULL --gebruik niet x=null 
>
> NOT (ontkenning)

transacties:

```plsql
begin; sql code; commit ;
begin; sql code; rollback;
start transaction; sql code; commit ;
start transaction; sql code; rollback;
```



# 2. Indexen & Optimalisaties



## Indexen

### Wat is een index?

Bertels zegt allemaal shit, maar hij kan niet effe uitleggen wat een index nu eigenlijk is. 

> Een index is een soort **lookup table**. De zoekmachine van je databank gebruikt deze om sneller dingen te vinden in je databank. Je kan het vergelijken met de index achteraan in een boek. 

**Voordeel**: index versnelt verwerking 

**Nadeel**: index neemt opslagruimte & elke mutatie vraagt aanpassing van index waardoor de verwerking vertraagt

:warning: een index can ook ineens corrupt worden blijkbaar. Dan krijg je ineens rare resultaten uit je query. Om het te fixen moet je je databank herindexeren.



### Soorten indexen

> Wie weet is dit nuttig, het komt rechtstreeks uit de slides



**Speciale indexvormen**

Multi-tabelindex : = index op kolommen in meerdere tabellen 

Virtuele-kolomindex : = index op een expressie 

Selectieve index = index op een gedeelte van de rijen 

Hash-index = index op basis van het adres op de pagina 

Bitmapindex = interessant als er veel dubbele waardes zijn



**Nieuwere indexvormen**

Standaard gebruik je een b-tree om een index aan te maken, er zijn ook andere manieren:

**GIN** (Generalized Inverted Index):

* Interessant bij veel dubbele waarden
* Er worden meerdere opzoekwaarden tegelijk aangemaakt (handig voor rij, tekst, json,..)
* Efficiënt bij dubbels, meerdere opzoekwaarden per veld; alternatief voor B-tree

**GiST** (Generalized Search Tree)

* template voor verschillende index schema’s
* Voor “clusters” volgens een afstandsmaat
* Bv vergelijken van intervallen, GIS, bevat, dichtste buren, tekst
* veel mogelijkheden, minder performant

**SP-GiST** (space-partitioned GiST)

* niet overlappende GiST

**BRIN**

* voor grote geclusterde tabellen
* bevat, grote geclusterde data, kleine index maar ook niet heel sterk



## Optimalisaties

Je kan die snelheid en kostprijs en nog allemaal andere leuke dingen over je query zien als je `EXPLAIN`  `EXPLAIN ANALYZE` voor je query zet.

Hoe kan je je queries nu zo efficient mogelijk maken?



**Een paar vuistregels:**

1. Vermijd de `OR`-operator

   > Dan wordt de index niet gebruikt. Je kan hem vervangen door een conditie met `IN` of door 2 selects met `UNION`.
   >
   > ```sql
   > where spelersnr in (15, 29, 55)
   > ```

2. Geen onnodig gebruik van `UNION`

   > Door `UNION` overloop je dezelfde tabel meerdere keren. Herformuleer je query en stop al je al je voorwaarden in één `SELECT` (indien mogelijk).

3. Vermijd de `NOT`-operator

   > `NOT` gebruikt ook de index niet. Herformuleer je code zodat hij alleen de vergelijkingsoperatoren gebruikt (<, >, =, ...)

4. Isoleer kolommen in condities

   > ik zal het schetsen met een voorbeeld
   >
   > ```sql
   > wherejaartoe + 10 = 1990 --omdat je die 10 erbij optelt wordt de index niet gebruikt
   > where jaartoe = 1980 --doe dit
   > ```

5. Gebruik de `BETWEEN`-operator

   > Want `AND` gebruikt de index meestal niet
   >
   > ```sql
   > where jaartoe >= 1985 and jaartoe <= 1990 --niet zo goed
   > where jaartoe between 1985 and 1990 --beter
   > ```

6. `LIKE` : index wordt niet gebruikt als patroon begint met % of _

7. Redundante condities bij joins

   > Om SQL te verplichten om een bepaald pad te kiezen
   >
   > ```sql
   > where boetes.spelersnr = spelers.spelersnr 
   > 	and boetes.spelersnr = 44
   > 	
   > where boetes.spelersnr = spelers.spelersnr --het is minder mooi, maar blijkbaar wel sneller
   > 	and boetes.spelersnr = 44 and spelers.spelersnr = 44 
   > ```

8. Vermijd de `HAVING`-component

   > `HAVING` gebruikt de index niet. Probeer indien mogelijk zo veel mogelijk condities in de `WHERE` te steken.

9. Maak de `SELECT`-component zo compact mogelijk

   > * Onnodige kolommen weglaten uit SELECT
   > * Bij gecorreleerde subquery met exists gebruik je best één expressie bestaande uit één constante, zie hieronder.
   >
   > ```sql
   > select spelersnr, naam
   >  from spelers
   >  where exists (select ‘1’
   >  		from boetes
   >  		where boetes.spelersnr = spelers.spelersnr)
   > ```

10. Vermijd `DISTINCT`

    > Je kan het vaak weglaten. `DISTINCT` verlengt de verwerkingstijd.

11. Gebruik de `ALL`-optie bij set operatoren

    > Zonder `ALL` is het trager omdat de data gesorteerd moet worden om alle dubbels eruit te halen.
    >
    > ```sql
    > Select naam, voorletters
    > from spelers
    > where spelersnr = 10
    > union all
    > select naam, voorletters
    > from spelers
    > where spelersnr = 18
    > ```

12. Kies outer-joins boven `UNION`

    > `UNION` verlengt de verwerkingstijd. 

13. Vermijd datatype-conversies

14. Zet de grootste tabel als laatste

15. Vermijd `ANY`- en `ALL`-operatoren

    > `ANY` en `ALL` gebruiken de index niet.
    >
    > ```sql
    > select spelersnr, naam, geb_datum
    > from spelers
    > where geb_datum <= all (select geb_datum from spelers)
    > --kan je omvormen naar
    > select spelersnr, naam, geb_datum
    > from spelers
    > where geb_datum = (select min(geb_datum) from spelers)
    > ```
    >
    > je kan ze vervangen door `min` of `max`.



# 3. Explain

Oh mijn fucking god *03_1_explain.pdf* is echt een teringzooi. Ik ga dit effe skippen want ik kan het niet aan.



## Dedection and benchmarking

### Statistieken

Er is een tool, genaamd **pg_stat_statements** waarmee je statistieken van je databank kan bekijken.

Als je hem wilt gebruiken moet je dit doen (in postgresql.conf):

```sql
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = all
--- en dan 
CREATE EXTENSION pg_stat_statements; -- in elke databank waar je het wilt gebruiken
```

### Benchmarking

Je kan **pgbench** gebruiken om je databank te benchmarken.

```sql
pgbench -i probeer
```

https://www.postgresql.org/docs/current/static/pgstatstatements.html 

https://www.postgresql.org/docs/current/static/pgbench.html



# 4. Limiting result sets & lateral

## Limiting result sets

Deze slides zijn ook weer heel onduidelijk, maar ik zal m'n best doen om het uit te leggen. Het doel van deze les is niet echt duidelijk in de slides. Het is heel simpel. We hebben een query, maar we willen maar een deel van de uitvoer tonen. Dit kan je met `OFFSET` en `FETCH`.

```sql
SELECT
FROM
..
ORDER BY
OFFSET start { ROW | ROWS }
FETCH { FIRST | NEXT } [ count ] { ROW | ROWS } ONLY
```

Stel je voor dat je alleen het derde tot het vijfde resultaat wilt tonen? 

Dan kan je bijvoorbeeld dit doen:

```sql
SELECT playernumber, name, birth_date
FROM players p
ORDER by birth_date DESC
OFFSET 2 ROWS
FETCH FIRST 3 ROWS ONLY;
```



## Lateral joins

Als je een subquery gebruikt in het `FROM` gedeelte, kan je in deze subqueries niet verwijzen naar de voorgaande kolommen die je met  `FROM` hebt opgeroepen. Dit kan je oplossen door `LATERAL` voor je subquery te zetten.

Zo werkt het dus niet:

```sql
SELECT
*,
(SELECT
sum(verblijfsduur),
variance(verblijfsduur)
FROM bezoeken b2
WHERE b2.reisnr = r1.reisnr
)
FROM reizen r1;
-- ERROR: subquery must return only one column
```

En zo wel:

```sql
SELECT *
FROM reizen r1 LEFT JOIN LATERAL
(SELECT
sum(verblijfsduur),
variance(verblijfsduur)
FROM bezoeken b2
WHERE b2.reisnr = r1.reisnr
) AS sub ON true;
```



# 5. Procedurele SQL

## PL/pgSQL

PL/pgSQL staat voor **Procedural Language/PostgreSQL**. Het is dus een procedurele programmeertaal (zoals Java) die je kan gebruiken in de databank.

Je kan functies opslaan in de database

```plsql
CREATE FUNCTION dup(in int, out f1 int, out f2 text)
 AS $$ SELECT $1, CAST($1 AS text) || ' is text' $$
 LANGUAGE SQL;
```

Je kan ze ook oproepen:

```sql
SELECT * FROM dup(42);
```

Algemeen:

```plsql
CREATE FUNCTION foo( arg1,arg2, ...)
RETURNS return_datatype
AS --block of code
LANGUAGE plpgsql;
```

Nog een simpel voorbeeld:

```plsql
CREATE OR REPLACE FUNCTION increment(i INT) RETURNS INT AS 
$$
  BEGIN
  RETURN i + 1;
  END;
$$ LANGUAGE 'plpgsql';
-- An example how to use the function:
SELECT increment(10);
```



## Functions vs Procedures

**Functies** geven een waarde terug, **procedures** niet. Het verschil is best vaag en hangt af van welke dbms je gebruikt. Ik zie het zo: Een functie is een stuk code dat een waarde geeft, een soort berekening. Terwijl een procedure echt letterlijk een procedure is, dus bijvoorbeeld iets in een tabel steken.

| Functies                                 | Procedures                          |
| ---------------------------------------- | ----------------------------------- |
| return                                   | geen return                         |
| Geen transacties                         | Wel transacties                     |
| Berekeningen                             | Manipulaties                        |
| Kan niet meerdere result sets teruggeven | Kan meerdere result sets teruggeven |
|                                          |                                     |



## Functions

Ik denk dat veel dingen die ik hier zet bij functions ook toepasbaar zijn op procedures. 



### Select into

> The `SELECT INTO` statement copies data from one table into a new table.

Dat kon Bertels niet even zeggen? Misschien moesten we dit al weten? Nou ja oké.

```plsql
create function som_boetes_speler(p_spelersnr integer) returns decimal(8,2) AS
$eenderwat$
 	declare
 	som_boetes decimal(8,2);
begin
 	select sum(bedrag)
 	into som_boetes
 	from boetes
 	where spelersnr = p_spelersnr;
 	return som_boetes ;
end
$eenderwat$ language ‘sql’;

select som_boetes_speler (27) ;
```



### Perform & Found

> Resultaten van een sql statement moeten opgevangen worden, bv via `INTO` variabele. `PERFORM` is een alternatief voor `SELECT` waarbij het resultaat niet wordt opgevangen

```plsql
PERFORM spelersnr FROM spelers ;
IF FOUND THEN .. END IF ;
```



### Exceptions

Je kan exceptions vangen en opgooien.

Vangen met `EXCEPTION`

```plsql
BEGIN
 -- code
 RAISE DEBUG 'A debug message % ', variable_that_will_replace_percent;
EXCEPTION
 -- welke fout, bv
 WHEN unique_violation THEN
 -- code
 WHEN division_by_zero THEN
 RAISE -- eventueel omzetten naar unique_violation;
 WHEN others THEN
 -- ?
 NULL;
END 
```

Opgooien met `RAISE`

```plsql
RAISE division_by_zero
--of
RAISE DEBUG/INFO/..
  SET client_min_messages TO debug;
--of
RAISE USING
 ERRCODE = 'unique_violation',
 HINT = 'suggestie voor de reden voor de gebruiker',
 DETAIL = 'meer detail fout',
 MESSAGE = 'gedrag van de unique_violation';
```



### Beveiliging

Je kan kiezen wie jouw functie mag aanpassen of uitvoeren.

```plsql
GRANT EXECUTE ON haha() TO jomeke ;
```



```plsql
CREATE OR REPLACE FUNCTION haha() RETURNS text AS
  $code$
    BEGIN
    DROP TABLE ola();
    RETURN ‘pola’;
    END
  $code$
LANGUAGE plpgsql
EXTERNAL SECURITY DEFINER;
```

In plaats van `DEFINER` kan je ook invoker zetten. 

> `SECURITY INVOKER` indicates that the function is to be executed with the privileges of the user that calls it. That is the default. `SECURITY DEFINER` specifies that the function is to be executed with the privileges of the user that created it.
>
> The key word `EXTERNAL` is allowed for SQL conformance, but it is optional since, unlike in SQL, this feature applies to all functions not only external ones.

[bron](https://www.postgresql.org/docs/9.5/sql-createfunction.html)



### Immutability

```plsql
CREATE FUNCTION add(integer, integer) RETURNS integer
 AS 'select $1 + $2;'
 LANGUAGE SQL
 IMMUTABLE
 RETURNS NULL ON NULL INPUT;
```

> `IMMUTABLE` indicates that the function cannot modify the database and always returns the same result when given the same argument values; that is, it does not do database lookups or otherwise use information not directly present in its argument list. If this option is given, any call of the function with all-constant arguments can be immediately replaced with the function value.
>
> `STABLE` indicates that the function cannot modify the database, and that within a single table scan it will consistently return the same result for the same argument values, but that its result could change across SQL statements. This is the appropriate selection for functions whose results depend on database lookups, parameter variables (such as the current time zone), etc. (It is inappropriate for `AFTER` triggers that wish to query rows modified by the current command.) Also note that the `current_timestamp` family of functions qualify as stable, since their values do not change within a transaction.
>
> `VOLATILE` indicates that the function value can change even within a single table scan, so no optimizations can be made. Relatively few database functions are volatile in this sense; some examples are `random()`, `currval()`, `timeofday()`. But note that any function that has side-effects must be classified volatile, even if its result is quite predictable, to prevent calls from being optimized away; an example is `setval()`.



## Procedures

Algemeen:

```plsql
CREATE OR REPLACE PROCEDURE
<procedure_name>( <arguments> )
AS <block-of-code>
LANGUAGE <implementation-language>;
```

Voorbeeldje:

```plsql
CREATE OR REPLACE PROCEDURE voorbeeld(invoer text )
AS $code$
	BEGIN
  IF invoer = ‘niet doen’ THEN
  RAISE WARNING 'Abort, the ship is sinking';
  ROLLBACK;
  IF invoer = ‘doen’ THEN
  RAISE INFO 'Gaon met die banaan';
  COMMIT;
	END IF;
END $code$ LANGUAGE plpgsql;
CALL voorbeeld(‘doen’);
```



## Triggers

Een **trigger** zorgt ervoor dat er bij een bepaalde gebeurtenis in de databank een stuk code opgeroepen wordt. Je kan bijvoorbeeld een kolom maken met een nummertje en elke keer dat er een insert wordt gedaan in je databank iets optellen bij dat nummertje.

Je kan triggers ook **functies** en **procedures** op laten roepen.

Algemeen, uit de [documentatie](https://www.postgresql.org/docs/9.1/sql-createtrigger.html):

```plsql
CREATE [ CONSTRAINT ] TRIGGER name { BEFORE | AFTER | INSTEAD OF } { event [ OR ... ] }
    ON table
    [ FROM referenced_table_name ]
    [ NOT DEFERRABLE | [ DEFERRABLE ] { INITIALLY IMMEDIATE | INITIALLY DEFERRED } ]
    [ FOR [ EACH ] { ROW | STATEMENT } ]
    [ WHEN ( condition ) ]
    EXECUTE PROCEDURE function_name ( arguments )

where event can be one of:
    INSERT
    UPDATE [ OF column_name [, ... ] ]
    DELETE
    TRUNCATE
```



```plsql
create trigger gebjaartoe
   before insert, update(geb_datum, jaartoe) of spelers
   for each row
   when (year(new.geb_datum) >= new.jaartoe)
   begin
   	rollback work ;
 end ;
```

Deze trigger checkt of de speler geboren is voor het jaar dat hij bij de tennisclub kwam. Als dat niet zo is voegt hij hem niet toe aan de tabel.

```plsql
create trigger delete_spelers
   after delete on spelers for each row
   begin
   delete from spelers_wed
   where spelersnr = old.spelersnr;
 end ;
```

Als er bij deze trigger een speler wordt verwijderd uit de tabel `spelers`, dan worden ook alle records met zijn nummer verwijderd uit de tabel `spelers_wed`.



# 6. GDPR

zou hij hier echt vragen over stellen?

Ik ga dit voor nu effe skippen.

Volgens Abel moet het erin. Oke dan.

## Hoofdpunten

* If data is related to EU citizens, the GDPR is applicable, independent of physical data location (AWS, Google, FB, …) 

* The right to have your personal data being erased completely 
* Only data on purpose can be hold by an organisation: of no informed consent or alternative use of data => penalties 
* Penalties can go up to max(20 mio€, 4% Annual Global Turnover) 
* Data Protection Officer (DPO) comes into play 
* It’s the organisation’s responsibility, not the one you outsourced your data to 
* Data necessary to run your business is OK (e.g. UCLL & your points earned)

## Database admins

* Admins will have to redesign their databases 
  * Privacy by Design & Default (PbD’s) 
    * Only store what you really need AND 
    * Give strict, (time) limited access to that data 
    * Opt-in will be the default choice for apps, not opt-out 
  * Make sure, one’s data can be erases quickly & completely 
  * Make sure, you log all usages of the data of someone => tracking breaches

## Info dissemination

* Local community (scouts, football, ..) owing e-mail addresses for corresponding time schedules & invoices cannot be used by default to invite people to a BBQ to raise money… 
* Informed consents ! 
* Checkbox forms that allow people to selectively do an optin with respect to the possible use of their personal data



## Application builders

* Privacy by Design & Default (PbD’s) 
  * Avoid use of personal data as much as possible on sites 
  * Ask for informed consent when storing personal data with clear annotation about the intended usage. 
  * Not only flowcharts for functionality, but also flowcharts to trace the personal data thru your app(lication): 
    * What kind of data ? 
    * Who got access ? 
    * What is it used for ?

## Data breaches

Data breach = a breach of security leading to accidental or unlawful destruction, loss, alteration, unauthorised disclosure or, or access to, personal data transmitted, stored or otherwise processed

* Automatic breach detection will become important since part of your Privacy by Design/Default 
* Notification of a breach has to be done within 72 hours timespan after discovery 
* More accurate reporting requested 
* You have to prove your good behaviour hence logs/tracking of data & access 
* You have to demonstrate the (limted) impact hence data flowcharts 
* You need to possess a plan to remedy the breach

Sorry, ik heb de slides gekopieerd. Als er nu een vraag komt, kan je dit als referentiemateriaal gebruiken.

# 7. Vensterfuncties

Vensterfuncties worden [hier](https://www.postgresql.org/docs/9.1/tutorial-window.html) best goed uitgelegd. Dan hoef ik het zelf niet te doen. Ik heb de stukken die ik nuttig vond hieronder geplakt. Als je een vraag krijgt hierover, raad ik aan om code uit *07_1_venster_functies.nld.pdf* te halen en wat aan te passen.

> A *window function* performs a calculation across a set of table rows that are somehow related to the current row. This is comparable to the type of calculation that can be done with an aggregate function. But unlike regular aggregate functions, use of a window function does not cause rows to become grouped into a single output row — the rows retain their separate identities. Behind the scenes, the window function is able to access more than just the current row of the query result.

```sql
SELECT depname, empno, salary, avg(salary) OVER (PARTITION BY depname) FROM empsalary;
```

```
	depname  | empno | salary |          avg          
-----------+-------+--------+-----------------------
 develop   |    11 |   5200 | 5020.0000000000000000
 develop   |     7 |   4200 | 5020.0000000000000000
 develop   |     9 |   4500 | 5020.0000000000000000
 develop   |     8 |   6000 | 5020.0000000000000000
 develop   |    10 |   5200 | 5020.0000000000000000
 personnel |     5 |   3500 | 3700.0000000000000000
 personnel |     2 |   3900 | 3700.0000000000000000
 sales     |     3 |   4800 | 4866.6666666666666667
 sales     |     1 |   5000 | 4866.6666666666666667
 sales     |     4 |   4800 | 4866.6666666666666667
(10 rows)
```

> The first three output columns come directly from the table `empsalary`, and there is one output row for each row in the table. The fourth column represents an average taken across all the table rows that have the same `depname` value as the current row. (This actually is the same function as the regular `avg` aggregate function, but the `OVER` clause causes it to be treated as a window function and computed across an appropriate set of rows.)

> A window function call always contains an `OVER` clause directly following the window function's name and argument(s). This is what syntactically distinguishes it from a regular function or aggregate function. The `OVER` clause determines exactly how the rows of the query are split up for processing by the window function. The `PARTITION BY` list within `OVER` specifies dividing the rows into groups, or partitions, that share the same values of the `PARTITION BY` expression(s). For each row, the window function is computed across the rows that fall into the same partition as the current row.

> You can also control the order in which rows are processed by window functions using `ORDER BY` within `OVER`. (The window `ORDER BY` does not even have to match the order in which the rows are output.) Here is an example:

Nog een voorbeeld van Bertels

```sql
SELECT row_number() OVER(ORDER BY spelersnr),
plaats, spelersnr
FROM spelers
WHERE geslacht = 'M'
ORDER BY plaats NULLS FIRST;
```

Dit geeft elke rij in de tabel een rijnummer volgens het spelersnr. Dus de speler met het laagste nummer krijgt 0, de volgende 1, ...



**RANK** geeft de cumulatieve frequentie. Kijk maar gewoon naar de uitvoer en dan snap je wel ongeveer wat het doet.

```sql
SELECT rank() OVER(ORDER BY plaats),
plaats
FROM spelers
WHERE geslacht = 'M'
ORDER BY plaats NULLS LAST;
```

```
  rank | plaats
 ------+----------
     1 | Den Haag
     1 | Den Haag
     1 | Den Haag
     1 | Den Haag
     1 | Den Haag
     1 | Den Haag
     1 | Den Haag
     8 | Rijswijk
     9 | Voorburg
(9 rows)
```

**DENSE RANK** (kijk maar gewoon)

```sql
SELECT dense_rank()
OVER(ORDER BY plaats),
plaats
FROM spelers
WHERE geslacht = 'M'
ORDER BY plaats NULLS LAST;
```

```
  dense_rank | plaats
 ------------+----------
           1 | Den Haag
           1 | Den Haag
           1 | Den Haag
           1 | Den Haag
           1 | Den Haag
           1 | Den Haag
           1 | Den Haag
           2 | Rijswijk
           3 | Voorburg
(9 rows)
```



# 8. CTEs en Beveiliging

## CTEs

**CTE**= common table expression

Een CTE is een tijdelijke set van resultaten waarnaar je vanuit een andere SQL statement zoals `SELECT` of `INSERT` kan verwijzen. Hiervoor gebruik je het woordje `WITH`.

```plsql
WITH cte_name (column_list) AS (
    CTE_query_definition 
)
--hier dan een query ofzo;
```

Met CTEs kan je complexe queries een stuk makkelijker maken. Je kan code-duplicatie vermijden en de leesbaarheid verbeteren. Nog iets leuks. Je kan er ook lussen mee maken.

```plsql
/*meer opties, boom tonen?*/
WITH RECURSIVE kind_van(bijnaam, vader, moeder) AS (
 SELECT bijnaam, vader, moeder
 FROM familieboom
 WHERE vader = 'vader'
 UNION ALL
 SELECT f.bijnaam, f.vader, f.moeder
 FROM familieboom f, kind_van k
 WHERE f.vader = k.bijnaam
)
SELECT bijnaam
FROM kind_van;
```

Als iemand zin heeft om dit uit te leggen: 1:moneybag:



### Oneindige lussen

Omdat je lussen kan maken met **CTEs**, kunnen er dus oneindige lussen voorkomen. Stel je nu voor, het schema *ruimtereizen*. Dat kennen jullie allemaal hopelijk al redelijk goed. Als je nu bij de Zon `satellietvan = aarde` zou doen en dan een CTE maakt die van elke planeet de naam print en dan verder gaat met zijn ouder (satellietvan). Dan zou in dit geval de lus oneindig door blijven lopen.

Er zijn **twee** om dit te voorkomen.

**Teller**

```plsql
WITH RECURSIVE kind_van(bijnaam, vader, moeder, diepte) AS (
  SELECT bijnaam, vader, moeder, 1
  FROM familieboom
  WHERE vader = 'vader'
   UNION ALL
  SELECT f.bijnaam, f.vader, f.moeder, diepte + 1 --teller incrementeren
  FROM familieboom f, kind_van k
  WHERE f.vader = k.bijnaam
  AND k.diepte<7 --bij teller == 7 stopt de lus
)
SELECT *
FROM kind_van;
```

**Lussen detecteren**

```plsql
WITH RECURSIVE kind_van(bijnaam, vader, moeder, pad, lus) AS (
 SELECT bijnaam, vader, moeder, ARRAY[vader] as pad, false
 FROM familieboom
 WHERE vader = 'vader'
 UNION ALL
 SELECT f.bijnaam, f.vader, f.moeder,
 CAST(k.pad || ARRAY[f.vader] as varchar(16)[]) as pad,
 f.vader = ANY(pad)
 FROM familieboom f, kind_van k
 WHERE f.vader = k.bijnaam
 AND NOT lus
)
SELECT *
FROM kind_van;
```

Zie de guide van deze les voor een meer *in depth* uitleg.



## Beveiliging

Oke dus je moet altijd je software up-to-date houden enzo. Binnen SQL kan je gebruik maken van **User management** (roles), **Privileges** (grant / revoke) , **Views** en **Stored procedures** om je databank veiliger te maken.

Je moet ook weten wat SQL injection is en hoe je je daartegen beschermt. 



# 9. ODMS, DBMS en ORDBMS

**ODBMS**: object(-oriented) database management system (is dit hetzelfde als ODMS?)

**ORD(BMS)**: object-relational database management system

**DBMS**: database management system



## ORDBMS

Je kan zelf types maken

```sql
CREATE type spelersnr AS integer;
CREATE TABLE spelers
(spelersnr spelersnr NOT null,
… );
DROP type spelersnr;
```

je kan ook machtigingen aan types geven

```plsql
revoke usage
on type geldbedrag
to Frank
```

je kan ook operator overloading doen

```plsql
bedrag1 + bedrag2 -- ongeldig
decimal(bedrag1) + decimal(bedrag2) -- mag wel
--je kan de '+' operator herdefineren om dit automatisch te doen.

create function « + » (geldbedrag, geldbedrag)
returns geldbedrag
source « + » (decimal(), decimal())
```

Voor meer info kan je een kijkje nemen in *09_3_ORDBMS.pdf*

Info over overerving en rules vind je ook hier.



### RULEs

> The **PostgreSQL rule** system allows one to define an alternative action to be performed on insertions, updates, or deletions in database tables. Roughly speaking, a **rule** causes additional commands to be executed when a given command on a given table is executed. 

[bron](https://www.postgresql.org/docs/9.2/sql-createrule.html)

Syntax:

```plsql
CREATE [ OR REPLACE ] RULE name AS ON event
    TO table_name [ WHERE condition ]
    DO [ ALSO | INSTEAD ] { NOTHING | command | ( command ; command ... ) }
```

Voorbeeld:

```plsql
CREATE RULE "NeKeerIetsAnders" AS
  ON SELECT TO wedstrijden
  WHERE spelersnr = 7
  DO INSTEAD
  SELECT 'eerst drie toerkes rond tafel
  lopen en dan nog eens
  proberen';
```



Wat is nu het verschil tussen een rule en een trigger?

> A **trigger** is fired **for** any affected row once. A **rule** manipulates the query or generates an additional query. So if many rows are affected in one statement, a **rule** issuing one extra command is likely to be faster than a **trigger** that is called for **every single row** and must execute its operations many times.

[bron](https://www.postgresql.org/docs/8.2/rules-triggers.html)

Nu moet ik toch wel zeggen dat de postgreSQL docs echt nuttig zijn.

## NoSQL

= Not only sql

Grote systemen zoals google en facebook gebruiken vaak NoSQL. 

> With NoSQL, you can:
>
> - Create documents without carefully defining their structure upfront
> - Add fields to your database without changing the fields of existing documents
> - Store documents that have their own unique structure
> - Have multiple databases with different structures and syntax

### ACID

NoSQL is ook niet echt ACID compliant.

> - Atomicity – each transaction either succeeds completely or is fully rolled back.
> - Consistency – data written to a database must be valid according to all defined rules.
> - Isolation – When transactions are run concurrently, they do not contend with each other, and act as if they were being run sequentially.
> - Durability – Once a transaction has been committed to the database, it is considered permanent, even in the event of a system failure.

Abel zegt dit

> Verband met NoSQL: NoSQL databanken zijn gemaakt om hoge beschikbaarheid te garanderen. Dit betekent dat ze vaak Consistency en Durability opofferen om dit te garanderen. Ze bieden wel Atomicity.
> Ze zijn ook niet altijd gesynchroniseerd, waardoor ze niet altijd consistent zijn. Maar ze komen uiteindelijk wel in sync, dus ze bieden wel deels consistency.

### CAP

Dan heb je ook het CAP-theorema. Ik weet niet echt hoe dit gelinkt wordt aan de leerstof, volgens Bertels worden 2 van de 3 voldaan door NoSQL.

> - *[Consistency](https://en.wikipedia.org/wiki/Consistency_model)*: Every read receives the most recent write or an error
> - *[Availability](https://en.wikipedia.org/wiki/Availability)*: Every request receives a (non-error) response, without the guarantee that it contains the most recent write
> - *[Partition tolerance](https://en.wikipedia.org/wiki/Network_partitioning)*: The system continues to operate despite an arbitrary number of messages being dropped (or delayed) by the network between nodes



# 10. XML en JSON

## XML

= extensible markup language

Jullie weten hopelijk al wat xml is.

```xml
<energie>
  <gerecht>
    <naam> frieten </naam>
    <quotering> zeer wel </quotering>
  </gerecht>
  <gerecht>
    <naam> sojapuddingzonderchoco </naam>
    <quotering> twijfelachtig </quotering>
  </gerecht>
</energie>
```

> xml (data) vs html (vorm) 
>
> ​			(xsl vs css)

Je kan ook metadata toevoegen in XML (bij dit voorbeeld de *type*)

```xml
<example type='music'> lalala.mp3 </example>
```

XML wordt voor verschillende dingen gebruikt:

* Data uitwisselen tussen applicaties
* Om de configuratie van je programma op te slaan
* HTML is ook een soort XML

### Voor- en nadelen

| Voordelen                       | Nadelen                                                      |
| ------------------------------- | ------------------------------------------------------------ |
| ISO-standaard                   | redundantie tov relationeel model, dus niet misbruiken in die zin |
| uitwisselbaarheid               | sequentieel, traag                                           |
| eenvoud                         |                                                              |
| geen hierarchische engine nodig |                                                              |



## XML en postgres

Ik ga hier gewoon wat XML functies en hun uitvoer droppen.

Zie de [XML Functions](https://www.postgresql.org/docs/11/functions-xml.html) documentatie voor meer info.

```sql
SELECT xmlcomment('hallo');
```

```xml
xmlcomment
--------------
<!--hallo-->
```



```sql
SELECT xmlelement(name jos,
 xmlattributes(current_date as ke), 'hal', 'lo');
```

```xml
xmlelement
-------------------------------------
<jos ke="2014-01-26">hallo</jos>
```



```sql
select xmlforest(divisie, teamnr) from teams; 
```

```xml
xmlforest
---------------------------------------------
<divisie>ere </divisie><teamnr>1</teamnr>
<divisie>tweede</divisie><teamnr>2</teamnr>
```



> The `xmlpi` expression creates an XML processing instruction. The content, if present, must not contain the character sequence `?>`.

```sql
SELECT xmlpi(name php, 'echo "Patat";');
```

```xml
xmlpi
-----------------------------
<?php echo "Patat";?>
```



> The following functions map the contents of relational tables to XML values. They can be thought of as XML export functionality:

```plsql
table_to_xml(tbl regclass, nulls boolean, tableforest boolean, targetns text)
query_to_xml(query text, nulls boolean, tableforest boolean, targetns text)
cursor_to_xml(cursor refcursor, count int, nulls boolean, tableforest boolean, targetns text)
```



## JSON

**JSON**: JavaScript Object Notation

Persoonlijk vind json beter dan xml. JSON is eigenlijk een simpelere versie van xml, het is leesbaarder en veel cooler.

<img src="https://pics.me.me/json-statham-62010142.png" alt="JSON Statham | Json Meme on ME.ME" width="30%;" />

Haha funni meme.



### JSON en postgres

Operators

| Operator | Right Operand Type | Return type       | Description                                                  | Example                                            | Example Result |
| -------- | ------------------ | ----------------- | ------------------------------------------------------------ | -------------------------------------------------- | -------------- |
| `->`     | `int`              | `json` or `jsonb` | Get JSON array element (indexed from zero, negative integers count from the end) | `'[{"a":"foo"},{"b":"bar"},{"c":"baz"}]'::json->2` | `{"c":"baz"}`  |
| `->`     | `text`             | `json` or `jsonb` | Get JSON object field by key                                 | `'{"a": {"b":"foo"}}'::json->'a'`                  | `{"b":"foo"}`  |
| `->>`    | `int`              | `text`            | Get JSON array element as `text`                             | `'[1,2,3]'::json->>2`                              | `3`            |
| `->>`    | `text`             | `text`            | Get JSON object field as `text`                              | `'{"a":1,"b":2}'::json->>'b'`                      | `2`            |
| `#>`     | `text[]`           | `json` or `jsonb` | Get JSON object at the specified path                        | `'{"a": {"b":{"c": "foo"}}}'::json#>'{a,b}'`       | `{"c": "foo"}` |
| `#>>`    | `text[]`           | `text`            | Get JSON object at the specified path as `text`              | `'{"a":[1,2,3],"b":[4,5,6]}'::json#>>'{a,2}'`      | `3`            |

er zijn er nog [meer](https://www.postgresql.org/docs/current/functions-json.html).

Deze zijn ook misschien nuttig om JSON aan te maken

* to_json(element) 
* array_to_json(n-dim array) 
* json_object (text[]) 
* json_object (keys text[], values text[])
*  …
* jsonb_* functions as well

Aggregatiefuncties:

* json_agg(expression) aggregates values as JSON array 
* json_object_agg(name, value) aggregates name/value pairs as a JSON object 
* jsonb_* functions as well

Specific processing functions

* json_array_length(json) 
* json_each(json) ~ pop 
* json_extract_path(x json, path_elems text[]): get specific elements 
* json_array_elements(json): array to values 
*  … 
* jsonb_* functions as well

Voorbeeldje

```sql
select * from json_each('{"a":"foo", "b":"bar"}') 

--uitvoer
 key | value
-----+-------
   a | “foo”
   b | “bar”
```



## Andere types

Er zijn ook nog andere types zoals vierkanten, lijnen, ip adressen, samengestelde types, ranges van getallen, ...

Als het echt nodig is kan je *10_3_OtherFormats.pdf* even lezen.



# Replicatie

> **Logical replication** is a method of **replicating** data objects and their changes, based upon their **replication** identity (usually a primary key). We use the term **logical** in contrast to physical **replication**, which uses exact block addresses and byte-by-byte **replication**.

> Logical replication uses a *publish* and *subscribe* model with one or more *subscribers* subscribing to one or more *publications* on a *publisher* node. Subscribers pull data from the publications they subscribe to and may subsequently re-publish data to allow cascading replication or more complex configurations.

[bron](https://www.postgresql.org/docs/10/logical-replication.html)

Het het dus **replicatie** omdat je de shizzle op de databank wilt repliceren. Stel je nu voor dat je een tweede databank hebt? Je zou graag willen dat die databank dezelfde inhoud bevat als de eerste (voor het geval er eentje spontaan ontploft). Dan kan je best gebruik maken van replicatie.



### Hoe doe je dat?

Op de eerste server maak je een publicatie

```plsql
CREATE PUBLICATION
```

Op de tweede maak je een subscription, je zal wel zelf de tabellen nog moeten aanmaken op deze databank (met dezelfde namen als in de eerste)

```sql
 CREATE SUBSCRIPTION
```



**Voorbeeld**

Op de eerste server (in dit geval databanken.ucll.be)

```plsql
ROLLBACK;
BEGIN;
CREATE schema replication_demo;
GRANT usage on SCHEMA replication_demo TO public;
SET search_path TO replication_demo;
CREATE TABLE demo_users(
 user_id serial PRIMARY KEY,
 first_name varchar(32) NOT NULL,
 last_name varchar(64)
);
GRANT select,insert ON demo_users TO public;
CREATE PUBLICATION mypub FOR TABLE demo_users;
--COMMIT;
```

Op de tweede (in dit geval jouw machine)

```plsql
CREATE schema replication_demo;
GRANT usage on SCHEMA replication_demo TO public;
SET search_path TO replication_demo;
CREATE TABLE demo_users(
 user_id serial PRIMARY KEY,
 first_name varchar(32) NOT NULL,
 last_name varchar(64)
);
CREATE SUBSCRIPTION mysub
CONNECTION 'dbname=probeer host=databanken.ucll.be user=local_u0082489
 port=51819 password=mypass sslmode=require' PUBLICATION mypub;
```



Ik leg dit normaal gezien uitvoerig uit in de guides

## Buffers

Bertels pull-up met de gepikte slides. Oh phew, gelukkig heeft hij de license erachter gezet. Ik had hem al bijna aangeklaagd.

Er staat ook echt niks ~~nuttigs~~ begrijpbaars in. Ik zou ook echt niet weten wat hij hierover zou kunnen vragen. Aarzel niet om een uitleg toe te voegen als je dat nuttig lijkt (:moneybag:).

