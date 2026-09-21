# Prometheus, Node exporter ja Grafana

## Johdanto

Monitorointi tarkoittaa jonkin prosessin tai järjestelmän jatkuvaa seurantaa ja havainnointia. Tietotekniikassa monitoroinnissa havainnoidaan esimerkiksi 
CPU-kuorman, verkkoliikenteen, muistin käytön ja suorituskyvyn muutoksia. Tarkoituksena ei ole kuitenkaan kerätä mahdollisimman paljon dataa, vaan kerätä sitä kohdennetusti tärkeimmistä kohteista riittävällä keräysvälillä. 

Monitorointi on tärkeää palveluiden ja verkkojen ylläpidossa, sillä ongelmat tulisi havaita ennen käyttäjän reagoimista niihin. Tarvitaan havainnointia normaalitasosta, jotta osataan reagoida poikkeamiin. Reaalikaisen muutoksen lisäksi seurataan siis myös historiallista dataa. Monitorointi on kokonaisuus, joka ulottuu datan keruusta aina laajemman kokonaiskuvan muodostamiseen ja analysoimiseen. Yhdistetään esimerkiksi eri mittareiden samanaikaista dataa. Monitoroinnin tarkoituksena on tukea päätöksentekoa.

Prometheus on avoimen lähdekoodin monitorointi- ja hälytystyökalu. Prometheus kerää ja tallentaa kaiken tiedon aikasarjadatana.  Kyseessä ei ole yksittäinen palvelin, vaan laaja ekosysteemi, joka koostuu erilaisita komponenteista, kuten Prometheus-palvelimesta, exportereista push gatewaysta. Prometheus käyttää tehokasta kyselykieltä PromQL 

## Ympäristön rakentaminen

Prometheus ei yleensä kerää tietoa suoraan käyttöjärjestelmältä, vaan käyttää exportereita. Exporterit, kuten Node exporter kerää käyttöjärjestelmältä tietoa, kuten CPU-, muisti- ja levyjärestelmän metriikoita ja julkaisee ne HTTP-rajapinnan kautta. 

### Node exporterin käyttöönotto

Kirjauduttiin aluksi web-palvelimelle komennolla **docker exec -it clab-hamk-verkonhallinta-golden-web1 bash**.
Ennen Node exporterin asennusta päivitettiin pakettiluettelot komennolla 
**sudo apt update**. Sen jälkeen asennettiin tarvittavat työkalut komennolla **apt install wget tar -y**. 
Wget mahdollistaa tiedostojen lataamisen internetistä ja tar mahdollistaa tar.gz-tiedostojen purkamisen.

Node exporterin uusin versio selvitettiin GitHubista ja asennettiin se komennolla:

**wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-1.12.1.linux-amd64.tar.gz**

Paketti purettiin komennolla:

**tar xvf node_exporter-1.12.1.linux-amd64.tar.gz**

Siirryttiin hakemistoon ja käynnistettiin Node exporter.

![Komentotuloste](images/node_exporter.png)

Node exporterin asentamisen ja käynnistämisen jälkeen tarkistettiin, että Node Exporter vastaa ja tarjoaa metriikoita pyynnöllä
 **curl http://localhost:9100/metrics**.

![Komentotuloste](images/metrics.png)

## Yhteyksien testaaminen

### Prometheus

Avattiin Prometheus selaimessa:
http://localhost:9090

Tarkistettiin Targets-näkymästä, että Prometheus pystyy hakemaan metriikoita web-palvelimelta web1. Tilana oli UP, joten yhteys toimii.

![Komentotuloste](images/prometheus_targets.png)

### Grafana

 Tarkistettiin myös Prometheuksen yhteys Grafanaan. Kirjauduttiin Grafanaan ja lisättiin tietolähteeksi (Data sources) Prometheus ja palvelimen osoitteeksi: http://prometheus:9090. Testattiin yhteyttä "save & test" -painikkeella ja saatiin ilmoitus onnistuneesta yhteydestä.

![Komentotuloste](images/grafanaquery.png)

## Dashboard

Luotiin dashboard nimeltä Golden Topology Monitoring ja lisättiin sinne paneelit, jotka näyttävät seuraavia metriikoita: CPU:n käyttöaste, muistin käyttöaste, levytilan käyttöaste, verkkoliikenne (saapuva) ja verkkoliikenne (lähtevä).

CPU:n käyttöaste kertoo kuinka paljon prosessorin laskentatehosta on käytössä. Mittari auttaa esimerkiksi paljastamaan CPU-pullonkaulat. Muistin käyttöaste kertoo käytössä olevan RAM:in osuuden kokonaismuistista. Sen on tärkeä toimintakyvyn mittari, sillä korkea muistin käyttöaste voi heikentää suorituskykyä tai johtaa järjestelmän kaatumiseen. Levytilan käyttöaste kertoo kuinka suuri osuus levytilasta on käytössä. Dashboardista huomataan, että verkkoliikenne kulkee nyt rajapinnan eht1 kautta ja etenkin saapuvassa liikenteessä huomataan muutosta, kun ajettiin komento **sudo apt install stress-ng** eli ladattiin paketit verkon kautta.

![Komentotuloste](images/dashboard.png)

### Kuormitustesti

Luotiin klo 16.14 500 megatavun testitiedosto komennolla **dd if=/dev/zero of=testfile.img bs=1M count=500**. Tiedoston luominen näkyy piikkinä levyntilan käyttöaste -paneelissa. Klo 16.06 piikki vuorostaan syntyi ajettuani komennon **sudo apt install stress-ng**. Kokonaiskuvan antavasta dashboardista voidaan havaita, että kuormitustesti ei aiheuta verkkoliikennettä.

![Komentotuloste](images/disk_usage.png)

Kuormitettiin CPU-ydintä 60 sekunnin ajan komennolla **stress-ng --cpu 4 --timeout 60s**. Komento ajettiin klo 16.16. Prosessorin kuormitus nousi hetkellisesti noin 6 prosenttiin. Kokonaiskuvan antavasta dashboardista voidaan havaita, että kuormitustesti ei aiheuta verkkoliikennettä.

![Komentotuloste](images/cpu_usage.png)

Muistin käyttöaste kasvoi hieman kuormitustestin aikana.

![Komentotuloste](images/memory_usage.png)



## SNMP vs Prometheus

|Ominaisuus|SNMP|Prometheus|
|:---------|:---|:---------|
|Tiedonkeruu||Perustuu pull-periaatteeseen (scraping). Eli kyselee metriikat pääsääntöisesti Node exporterilta. |
|Käyttöönotto|||
|Mittarien määrä|||
|Visualisointi||Ei ole itsessään visualisointityökalu, mutta toimii hyvin mittausdatan visualisointityökalun Grafanan kanssa|
|Hälytysmahdollisuudet||Hälytystyökalu (Alert Manager) kuuluu ekosysteemiin|
|Soveltuvuus pilviympäristöihin||Soveltuu hyvin pilviympäristöihin, palvelin- ja konttiympäristöjen lisäksi|

## Pohdinta

Keskeisimmät mittarit, mitä ylläpitäjän tulisi seurata liittyvät kohteen saatavuuteen, resurssien käyttöön ja verkkoliikenteeseen. Ensinnäkin on hyvä tietää saadanko kohteesta dataa, eli onko kohteen tila ylhäällä (up). Resurssien käytöstä kertovat esimerkiksi CPU:n ja muistin käyttöaste. Ne ovat tärkeitä toimintakyvyn mittareita, jotka kertovat sen hetkisestä järjestelmien kuormituksesta. Jatkuva korkea muistin tai CPU:n käyttöaste voi kertoa ongelmasta tai optimoinnin tarpeesta. Sen lisäksi voidaan seurata levytilan käyttöä ja varmistaa, että levytilaa on riittävästi. Verkkoliikennettä seurattiin harjoituksessa lähtevän (transmit) ja saapuvan (receive) liikenteen osalta. Myös nämä kuuluvat keskeisimpiin mittareihin, joilla varmistetaan verkon toimivuus. Prometheuksen dokumentaatiossa korostetaan, että  metriikoiden valinta riippuu aina käyttötarkoituksesta. Web-palvelimen kohdalla voi olla tärkeää seurata kuinka nopeasti palvelin vastaa pyyntöihin (request time) tai kuinka monta pyyntöä vastaanotetaan (request rate), kun taas tietokannan kohdalla voidaan seurata aktiivisten kyselyiden määrää.

Tehtävässä rakennettu dashboard antaa perustiedot infrastruktuurin terveydestä, mutta lisäisin siihen ainakin virheistä


Mitä hyötyä Prometheuksesta on verrattuna SNMP:hen?
Millaisia mittareita ylläpitäjän kannattaa seurata jatkuvasti?
Mitä tietoa dashboardisi tarjoaa ylläpitäjälle?
Mitä uusia mittareita lisäisit dashboardiin?
Miten monitorointitiedosta voisi olla hyötyä vianetsinnässä?





 

