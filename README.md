# Projekt z przedmiotu Technologie nosql

### Wybrany przeze mnie zbiór danych:

Nazwa zbioru: **_Twitter Data For Sentiment Analysis_**

Wielkość zbioru: **_1 600 000 tweetów_**

Rozmiar zbioru: **_233 207KB_**

Źródło danych: **_[kliknij mnie!](http://help.sentiment140.com/for-students/)_**



### Parametry komputera testowego

|Jednostka|Parametr|
|------------|:-------------:|
|System|Windows 7 32bit|
|Procesor|Intel(R) Core(TM)2 Quad CPU|
|Ilość rdzeni|4|
|Moc rdzenia|2.66GHz|
|Pamięć RAM|4GB(3.25GB dostępne)|

### Przedstawienie danych

Przykładowy rekord:

``` json
{
	"Rating": "0",
	"id": "1467812416",
	"CreationData": "Mon Apr 06 22:20:16 PDT 2009",
	"Username": "erinx3leannexo",
	"Tweet": "spring break in plain city... it's snowing ",
	"Location": [-115.1372200,36.1749700]
}
```

#### Wyjaśnienie pól

* Rating - ocena tweeta( 0 - negaytwna, 4 - pozytywna )
* id - numer id tweeta
* CreationData - data udostępnienia tweeta
* Username - nazwa użytkownika tweeta
* Tweet - tekst tweeeta
* Latitude - szerokość geograficzna
* Longitude - długość geograficzna

### Przykładowe zapytania

Informacje, których będę szukał to np. najaktywniejsi użytkownicy, miejsca, z których zostało wysłanych najwięcej tweetów czy statystyka miesięczna( zbiór zawiera tweety z 3 miesięcy 2009 roku ).

Przykład zapytania w PostgreSQL( 5 najaktywniejszych użytkowników ):

```SELECT username, COUNT(*) AS tweets FROM schema.tweets GROUP BY username ORDER BY tweets DESC LIMIT 5```

### Czyszczenie danych

[Plik](https://docs.google.com/uc?id=0B04GJPshIjmPRnZManQwWEdTZjg&export=download) zawierający dane nie zawierał nagłówków, a opis poszczególnych kolumn znajdował się na [stronie](http://help.sentiment140.com/for-students/), z której został pobrany zbiór. Postanowiłem usunąć kolumnę 4-tą zawierającą zapytania do postów, ponieważ we wszystkisch rekordach była ta sama wartość "NO_QUERY", która nic nie wnosiła do danych. Dodatkowo postanowiłem o dodaniu lokalizacji tych tweetów wprowadzając listę współrzędnych 31 miejscowości USA i losując( programistycznie ) do każdego tweeta jedną z nich.

## Zadanie GEO

Ponieważ wybrany przeze mnie zbiór( tweety ) nie zawierał danych geolokalizacji, musiałem je dołączyć programistycznie. Zrobiłem to poprzez dolosowanie dla każdego tweeta geolokalizacji 1 z 31 wybranych przeze mnie miast USA. Jako, iż tych geolokalizacji jest tylko 31 dla 1 600 000 tweetów( byłoby mnóstwo znaczników w 1 miejscu ), do tego zadania postanowiłem zliczyć ilość wystąpień każdej z lokalizacji i utworzenie osobnego zbioru zawierającego dane:

||Country|City|Tweets|Location|
|---|------|---|-----------|-----|
|typ|text|text|long|geo_point|

* Country - kraj, z którego został wysłany tweet
* City - miasto, z którego został wysłany tweet
* Tweets - ilość tweetów wysłanych z tego miejsca
* Location - współrzędne geograficzne

![alt tag](https://github.com/kropeq/nosql/blob/master/screens/mapka_geojson.png)

#### Utworzenie bazy

```curl -XPUT localhost:9200/geo```

#### Dodanie mappingu

```curl -XPUT localhost:9200/geo/_mapping/cities --data-binary @mapping_geo.json```

#### Import danych

```curl -XPOST localhost:9200/geo/cities/_bulk --data-binary @tweets_geo.json```

### Rozwiązania w postaci mapek są dostępne pod tym linkiem: [https://kropeq.github.io/](https://kropeq.github.io/)

#### Zapytanie 1: Miejsca tweetujących w odległości 400 kilometrów od miasta Baltimore

``` curl -g -X GET "http://localhost:9200/geo/cities/_search?pretty=true" --data-binary @query1.txt```

#### query1.txt:

```
{
"query": { 
	"bool" : { 
		"must" : {
			"match_all" : {} 
			},
		"filter" : { 
			"geo_distance" : { 
				"distance" : "400km", 
				"Location": [-76.6121900,39.2903800] 
				}
			}
		}
	}
}
```

#### Zapytanie 2: Miejsca tweetujących w stanie Teksas

``` curl -g -X GET "http://localhost:9200/geo/cities/_search?pretty=true" --data-binary @query2.txt```

#### query2.txt:

```
{
"query": { 
	"bool" : { 
		"must" : {
			"match_all" : {} 
			},
		"filter" : { 
			"geo_polygon" : { 
				"Location" : {
					"points" : [
						[-97.450601, 26.395055],
						[-106.600256, 31.972835],
						[-103.061962, 31.996343],
						[-103.034247, 36.499653],
						[-100.004064, 36.484799],
						[-99.967110, 34.567728],
						[-94.072946, 33.550980],
						[-93.858713, 29.693271]
					]
				}
			}
		}
	}
}
}
```

#### Zapytanie 3: Miejsca tweetujących północno-wschodniej części USA

``` curl -g -X GET "http://localhost:9200/geo/cities/_search?pretty=true" --data-binary @query3.txt```

#### query3.txt:

```
{
"query": { 
	"bool" : { 
		"must" : {
			"match_all" : {} 
			},
		"filter" : { 
			"geo_bounding_box" : {
				"Location" : {
					"top_left": [-89.635546,43.709397],
					"bottom_right": [-68.992815,37.445628]
				}
			}
		}
	}
}
}
```

## PostgreSQL

### Zadanie 1 ( EDA )

Dane: _[Pobierz](https://docs.google.com/uc?id=0B04GJPshIjmPRnZManQwWEdTZjg&export=download)_ (Twitter Data For Sentiment Analysis)

Plik: _training.1600000.processed.noemoticon.csv_

##### Wstawiamy plik _training.1600000.processed.noemoticon.csv_ do wspólnego folderu z plikiem _edycja_csv.class_ i z linii poleceń uruchamiamy konwersję:

```java -cp źródło\pliku edycja_csv```

#### W wyniku otrzymujemy plik tweets.csv, który zawiera "oszczyszczone" dane, okrojoną liczbę kolumn i dodaną geolokalizację. Kolumny to kolejno:

|Rating|Id|CreationData|Username|Tweet|Latitude|Longitude|
|------|---|-----------|--------|-----|--------|---------|

#### Łączenie się z postgresem

```psql -U postgres -h localhost```

#### Uruchomienie pomiarów czasowych

```\timing```

#### Tworzenie bazy danych baza

```CREATE database baza```

#### Czas tworzenia bazy

```5,533938s```

#### Łączenie z bazą danych

```\c baza```

#### Tworzenie schematu

```CREATE SCHEMA schema```

#### Czas tworzenia schematu

```0,000694s```

#### Tworzymy tabelę tweets

```CREATE TABLE schema.tweets(Rating integer, Id bigint, CreationDate date, Username varchar, Tweet varchar, Latitude numeric, Longitude numeric)```

#### Czas tworzenia tabeli

```0,267083s```

#### Importowanie danych do tabeli z przygotowanego pliku

```\copy schema.tweets FROM 'C:/Users/MINIO/Desktop/TEST/tweets.csv' DELIMITER ',' CSV HEADER```

#### Czas importowanie danych

```20,282551s```

#### Zliczenie ilości danych

```SELECT COUNT(*) FROM schema.tweets```

#### Czas zliczania danych

```1,026159s```

#### Liczba danych

```1 600 000```

#### Znalezienie 5 najbardziej aktwynych użytkowników

```SELECT username, COUNT(*) AS tweets FROM schema.tweets GROUP BY username ORDER BY tweets DESC LIMIT 5```

#### Czas szukania i wyliczania

```29,315564s```

#### Zwrócony wynik:

|username|tweets|
|------------|:-------------:|
|lost_dog|549|
|webwoke|345|
|tweetpet|310|
|SallytheShizzle|281|
|VioletsCRUK|279|

#### Sprawdzanie, w którym miesiącu roku( 2009 ) było najwięcej tweetów

```SELECT extract(month FROM creationdate) as month, COUNT(*) as tweets FROM schema.tweets GROUP BY 1 ORDER BY tweets DESC```

#### Czas sprawdzania

```1,529245s```

#### Zwrócony wynik

|month|tweets|
|------------|:-------------:|
|6|923608|
|5|576367|
|4|100025|

#### Sprawdzanie, w którym miesiącu roku( 2009 ) było najwięcej tweetów z nazwą miesiąca

```SELECT to_char(to_timestamp(to_char(extract(month FROM creationdate),'999'),'MM'),'Month') as month, COUNT(*) as tweets FROM schema.tweets GROUP BY 1 ORDER BY tweets DESC```

#### Czas sprawdzania

```10,160789s```

#### Zwrócony wynik

|month|tweets|
|------------|:-------------:|
|June|923608|
|May|576367|
|April|100025|


## Elasticsearch

Do pomiarów czasu wykorzystuję skrypt [timecmd.bat](http://stackoverflow.com/questions/673523/how-to-measure-execution-time-of-command-in-windows-command-line)

Operacje będę wykorzystywał do przygotowanej próbki danych( 1000 rekordów ) w formacie json.

#### Wstawiam plik _training.1600000.processed.noemoticon.csv_ do wspólnego folderu z plikiem _csv_to_json.class_ i z linii poleceń uruchamiam konwersję:

```java -cp źródło\pliku csv_to_json```

#### W wyniku otrzymuje plik tweets_small.json, który zawiera "oszczyszczone" dane, okrojoną liczbę kolumn i dodaną geolokalizację. Kolumny to kolejno:

|   |Rating|Id|CreationData|Username|Tweet|Location|
|---|------|---|-----------|--------|-----|---------|
|Typ|long|long|date|text|text|geo_point|

#### Przygotowałem mapping do mojej bazy

[mapping](https://github.com/kropeq/nosql/blob/master/mapping.json)

```
{
  	"tweet": {
		"properties": {
			"Rating": { "type": "long" },
			"id": { "type": "long" },
			"CreationData": { "type": "date", "format": "EEE MMM dd HH:mm:ss z yyyy" },
			"Username": { "type": "text"},
			"Tweet": { "type": "text" },
			"Location": { "type":"geo_point" }
		}
	}
}
```

#### Utworzenie indexu "tweets"

```timecmd curl -XPUT localhost:9200/tweets```

```0.22s```

#### Ustawiamy mapping typu "tweet"

```timecmd curl -XPUT localhost:9200/tweets/_mapping/tweet --data-binary @mapping.json```

```0.09s```

#### Import pliku z danymi

```timecmd curl -XPOST localhost:9200/tweets/tweet/_bulk --data-binary @tweets.json```

```0.53s```

#### Zliczenie zaimportowanych danych

```timecmd curl localhost:9200/tweets/tweet/_count```

```"count":1000```

```0.14s```



## MongoDB

Formaty DateTime i liczb powinny być poprawnie zaimportowane.(TODO)

### Zadanie 2 ( EDA )

Dane: _[Pobierz](https://docs.google.com/uc?id=0B04GJPshIjmPRnZManQwWEdTZjg&export=download)_ (Twitter Data For Sentiment Analysis)

Plik: _training.1600000.processed.noemoticon.csv_

##### Wstawiamy plik _training.1600000.processed.noemoticon.csv_ do wspólnego folderu z plikiem _edycja_csv.class_ i z linii poleceń uruchamiamy konwersję:

```java -cp źródło\pliku edycja_csv```

#### W wyniku otrzymujemy plik tweets.csv, który zawiera "oszczyszczone" dane, okrojoną liczbę kolumn i dodaną geolokalizację. Kolumny to kolejno:

|Id|CreationData|Username|Tweet|Latitude|Longitude|
|---|-----------|--------|-----|--------|---------|

#### Tworzenie bazy danych w MongoDB:

```use baza```

#### Import danych z przygotowanego pliku CSV z pomiarem czasu

```powershell "Measure-Command{mongoimport -d baza -c tweets --type csv --file C:\folder\tweets.csv --headerline}"```

#### Czas importu

```Total seconds : 73,2462208```

##### Liczba zaimportowanych danych

```1 600 000```







## Zadania, które później zniknęły z listy zadań

### Zadanie 1 POSTGRES

Dane: _[Pobierz](https://archive.org/download/stackexchange/gamedev.stackexchange.com.7z)_ (Game Development Posts)

Plik: _Posts.xml_

##### Wstawiamy plik _Posts.xml_ do wspólnego folderu z plikiem _xml_to_csv.class_ i z linii poleceń uruchamiamy konwersję:

```java -cp źródło\pliku xml_to_csv```

#### W wyniku otrzymujemy plik posts.csv, który zawiera "oszczyszczone" dane, a także okrojoną liczbę kolumn. Kolumny to kolejno:

|Id|ViewCount|CreationData|Body|Title|Tags|AnswerCount|CommentCount|
|---|--------|------------|----|-----|----|-----------|------------|

#### Łączenie się z postgresem

```psql -U postgres -h localhost```

#### Uruchomienie pomiarów czasowych

```\timing```

#### Tworzenie bazy danych baza

```CREATE database baza```

#### Czas tworzenia bazy

```5,533938s```

#### Łączenie z bazą danych

```\c baza```

#### Tworzenie schematu

```CREATE SCHEMA schema```

#### Czas tworzenia schematu

```0,000694s```

#### Tworzenie tabeli posts

```CREATE TABLE schema.posts( Id integer, ViewCount integer, CreationDate date, Body varchar, Title varchar, Tags varchar, AnswerCount integer, CommentCount integer)```

#### Czas tworzenia tabeli

```0,214664s```

#### Import danych

```\copy schema.posts FROM 'C:/Users/MINIO/Desktop/TEST/posts.csv' DELIMITER ';' CSV HEADER```

#### Czas importowania

```10,092721s```

#### Liczba danych

```SELECT COUNT(*) FROM schema.posts```

#### Liczba danych

```94185```

#### Czas zliczania rekordów tabeli

```0,089496s```



### Zadanie 1 MONGODB

Dane: _[Pobierz](https://archive.org/download/stackexchange/gamedev.stackexchange.com.7z)_ (Game Development Posts)

Plik: _Posts.xml_

##### Wstawiamy plik _Posts.xml_ do wspólnego folderu z plikiem _xml_to_csv.class_ i z linii poleceń uruchamiamy konwersję:

```java -cp źródło\pliku xml_to_csv```

#### W wyniku otrzymujemy plik posts.csv, który zawiera "oszczyszczone" dane, a także okrojoną liczbę kolumn. Kolumny to kolejno:

|Id|ViewCount|CreationData|Body|Title|Tags|AnswerCount|CommentCount|
|---|--------|------------|----|-----|----|-----------|------------|

#### Tworzenie bazy danych w MongoDB:

```use baza```

#### Import danych z przygotowanego pliku CSV z pomiarem czasu

```powershell "Measure-Command{mongoimport -d baza -c posts --type csv --file C:\folder\posts.csv --headerline}"```

#### Czas importu

```Total seconds : 13,1237809```

##### Liczba zaimportowanych danych

```94185```
