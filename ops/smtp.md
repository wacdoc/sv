# Bygg din egen SMTP-postsändningsserver

## inledning

SMTP kan köpa tjänster direkt från molnleverantörer, såsom:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali moln e-post push](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Du kan också bygga din egen e-postserver - obegränsad sändning, låg total kostnad.

Nedan visar vi steg för steg hur man bygger vår egen e-postserver.

## Serverval

Den självvärdade SMTP-servern kräver en offentlig IP med portarna 25, 456 och 587 öppna.

Vanligt använda offentliga moln har blockerat dessa portar som standard, och det kan vara möjligt att öppna dem genom att utfärda en arbetsorder, men det är trots allt väldigt besvärligt.

Jag rekommenderar att du köper från en värd som har dessa portar öppna och som stöder inställning av omvända domännamn.

Här rekommenderar jag [Contabo](https://contabo.com) .

Contabo är en webbhotell baserad i München, Tyskland, grundad 2003 med mycket konkurrenskraftiga priser.

Om du väljer Euro som inköpsvaluta blir priset billigare (en server med 8 GB minne och 4 processorer kostar cirka 529 yuan per år, och den initiala installationsavgiften är gratis i ett år).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

När du gör en beställning, anmärkning `prefer AMD` , och servern med AMD CPU kommer att ha bättre prestanda.

I det följande ska jag ta Contabos VPS som exempel för att demonstrera hur man bygger sin egen mailserver.

## Ubuntu systemkonfiguration

Operativsystemet här är Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Om servern på ssh visar `Welcome to TinyCore 13!` (som visas i figuren nedan), betyder det att systemet inte har installerats ännu. Koppla från ssh och vänta i några minuter för att logga in igen.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

När `Welcome to Ubuntu 22.04.1 LTS` visas är initieringen klar och du kan fortsätta med följande steg.

### [Valfritt] Initiera utvecklingsmiljön

Detta steg är valfritt.

För enkelhetens skull lägger jag installationen och systemkonfigurationen av ubuntu-programvaran i [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Kör följande kommando för att installera med ett klick.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Kinesiska användare, använd följande kommando istället, så ställs språket, tidszonen etc. in automatiskt.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo möjliggör IPV6

Aktivera IPV6 så att SMTP även kan skicka e-postmeddelanden med IPV6-adresser.

redigera `/etc/sysctl.conf`

Ändra eller lägg till följande rader

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Följ upp med [kontakthandledningen: Lägga till IPv6-anslutning till din server](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Redigera `/etc/netplan/01-netcfg.yaml` , lägg till några rader som visas i figuren nedan (Contabo VPS standardkonfigurationsfil har redan dessa rader, bara avkommentera dem).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

`netplan apply` för att få den ändrade konfigurationen att träda i kraft.

När konfigurationen har lyckats kan du använda `curl 6.ipw.cn` för att se ipv6-adressen för ditt externa nätverk.

## Klona ops för konfigurationslagret

```
git clone https://github.com/wactax/ops.soft.git
```

## Skapa ett gratis SSL-certifikat för ditt domännamn

För att skicka e-post krävs ett SSL-certifikat för kryptering och signering.

Vi använder [acme.sh](https://github.com/acmesh-official/acme.sh) för att generera certifikat.

acme.sh är ett automatiserat certifikatsigneringsverktyg med öppen källkod,

Gå in i konfigurationslagret ops.soft, kör `./ssl.sh` och en `conf` mapp kommer att skapas i **den övre katalogen** .

Hitta din DNS-leverantör från [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , redigera `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Kör sedan `./ssl.sh 123.com` för att generera `123.com` och `*.123.com` certifikat för ditt domännamn.

Den första körningen kommer automatiskt att installera [acme.sh](https://github.com/acmesh-official/acme.sh) och lägga till en schemalagd uppgift för automatisk förnyelse. Du kan se `crontab -l` , det finns en sådan rad som följer.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Sökvägen för det genererade certifikatet är ungefär `/mnt/www/.acme.sh/123.com_ecc。`

Certifikatförnyelse kommer att anropa `conf/reload/123.com.sh` script, redigera detta skript, du kan lägga till kommandon som `nginx -s reload` för att uppdatera certifikatcachen för relaterade applikationer.

## Bygg SMTP-server med chasquid

[chasquid](https://github.com/albertito/chasquid) är en SMTP-server med öppen källkod skriven på Go-språket.

Som ett substitut för de gamla e-postserverprogrammen som Postfix och Sendmail är chasquid enklare och enklare att använda, och det är också lättare för sekundär utveckling.

Kör `./chasquid/init.sh 123.com` kommer att installeras automatiskt med ett klick (ersätt 123.com med ditt avsändande domännamn).

## Konfigurera e-postsignatur DKIM

DKIM används för att skicka e-postsignaturer för att förhindra att brev behandlas som skräppost.

När kommandot har körts kommer du att uppmanas att ställa in DKIM-posten (som visas nedan).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Lägg bara till en TXT-post till din DNS (som visas nedan).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Visa tjänstens status och loggar

 `systemctl status chasquid` Visa tjänstens status.

Tillståndet för normal drift är som visas i figuren nedan

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` eller `journalctl -xeu chasquid` kan se felloggen.

## Omvänd domännamnskonfiguration

Det omvända domännamnet är för att tillåta IP-adressen att lösas till motsvarande domännamn.

Att ställa in ett omvänt domännamn kan förhindra att e-postmeddelanden identifieras som skräppost.

När e-posten tas emot kommer den mottagande servern att utföra omvänd domännamnsanalys på den sändande serverns IP-adress för att bekräfta om den sändande servern har ett giltigt omvänt domännamn.

Om den sändande servern inte har ett omvänt domännamn eller om det omvända domännamnet inte matchar IP-adressen för den sändande servern, kan den mottagande servern känna igen e-postmeddelandet som skräppost eller avvisa det.

Besök [https://my.contabo.com/rdns](https://my.contabo.com/rdns) och konfigurera enligt nedan

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Efter att ha ställt in det omvända domännamnet, kom ihåg att konfigurera vidareupplösningen för domännamnet ipv4 och ipv6 till servern.

## Redigera värdnamnet för chasquid.conf

Ändra `conf/chasquid/chasquid.conf` till värdet för det omvända domännamnet.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Kör sedan `systemctl restart chasquid` för att starta om tjänsten.

## Säkerhetskopiera conf till git repository

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Till exempel säkerhetskopierar jag conf-mappen till min egen github-process enligt följande

Skapa ett privat lager först

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Gå in i conf-katalogen och skicka till lagret

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Lägg till avsändare

springa

```
chasquid-util user-add i@wac.tax
```

Kan lägga till avsändare

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Kontrollera att lösenordet är korrekt inställt

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Efter att ha lagt till användaren kommer `chasquid/domains/wac.tax/users` att uppdateras, kom ihåg att skicka in det till lagret.

## DNS lägg till SPF-post

SPF ( Sender Policy Framework ) är en e-postverifieringsteknik som används för att förhindra e-postbedrägeri.

Den verifierar identiteten för en e-postavsändare genom att kontrollera att avsändarens IP-adress stämmer överens med DNS-posterna för domännamnet den påstår sig vara, vilket förhindrar bedragare från att skicka falska e-postmeddelanden.

Att lägga till SPF-poster kan förhindra att e-postmeddelanden identifieras som skräppost så mycket som möjligt.

Om din domännamnsserver inte stöder SPF-typ, lägg bara till TXT-typpost.

Till exempel är SPF för `wac.tax` som följer

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF för `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Observera att jag har `include:_spf.google.com` här, det beror på att jag kommer att konfigurera `i@wac.tax` som avsändningsadress i Googles brevlåda senare.

## DNS-konfiguration DMARC

DMARC är förkortningen av (Domain-based Message Authentication, Reporting & Conformance).

Den används för att fånga SPF-avvisningar (kanske orsakade av konfigurationsfel eller att någon annan utger sig för att vara du för att skicka skräppost).

Lägg till TXT-post `_dmarc` ,

Innehållet är som följer

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Betydelsen av varje parameter är följande

### p (policy)

Indikerar hur man hanterar e-postmeddelanden som misslyckas med SPF (Sender Policy Framework) eller DKIM (DomainKeys Identified Mail) verifiering. Parametern p kan ställas in på ett av tre värden:

* ingen: Ingen åtgärd vidtas, endast verifieringsresultatet återkopplas till avsändaren via e-postrapporteringsmekanismen.
* Karantän: Lägg posten som inte har klarat verifieringen i skräppostmappen, men kommer inte att avvisa posten direkt.
* avvisa: Avvisa e-postmeddelanden som misslyckas med verifiering direkt.

### fo (felalternativ)

Anger mängden information som returneras av rapporteringsmekanismen. Den kan ställas in på ett av följande värden:

* 0: Rapportera valideringsresultat för alla meddelanden
* 1: Rapportera endast meddelanden som misslyckas med verifiering
* d: Rapportera endast fel på domännamnsverifiering
* s: rapportera endast SPF-verifieringsfel
* l: Rapportera endast DKIM-verifieringsfel

### rua & ruf

* rua (Rapporterings-URI för samlade rapporter): E-postadress för att ta emot sammanställda rapporter
* ruf (Reporting URI for Forensic reports): e-postadress för att få detaljerade rapporter

## Lägg till MX-poster för att vidarebefordra e-post till Google Mail

Eftersom jag inte kunde hitta en gratis företagspostlåda som stöder universella adresser (Catch-All, kan ta emot alla e-postmeddelanden som skickas till detta domännamn, utan begränsningar för prefix), använde jag chasquid för att vidarebefordra alla e-postmeddelanden till min Gmail-postlåda.

**Om du har din egen betalda företagspostlåda, vänligen ändra inte MX och hoppa över det här steget.**

Redigera `conf/chasquid/domains/wac.tax/aliases` , ställ in postlåda för vidarebefordran

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` indikerar alla e-postmeddelanden, `i` är e-postadressprefixet för den avsändande användaren som skapats ovan. För att vidarebefordra e-post måste varje användare lägga till en rad.

Lägg sedan till MX-posten (jag pekar direkt på adressen till det omvända domännamnet här, som visas på första raden i figuren nedan).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

När konfigurationen är klar kan du använda andra e-postadresser för att skicka e-post till `i@wac.tax` och `any123@wac.tax` för att se om du kan ta emot e-postmeddelanden i Gmail.

Om inte, kontrollera chasquid-loggen ( `grep chasquid /var/log/syslog` .

## Skicka ett e-postmeddelande till i@wac.tax med Google Mail

Efter att Google Mail tagit emot mailet hoppades jag naturligtvis på att svara med `i@wac.tax` istället för i.wac.tax@gmail.com.

Besök [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) och klicka på "Lägg till en annan e-postadress".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Ange sedan verifieringskoden som mottogs av e-postmeddelandet som vidarebefordrades till.

Slutligen kan den ställas in som standardavsändaradress (tillsammans med möjligheten att svara med samma adress).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

På så sätt har vi slutfört etableringen av SMTP-postservern och använder samtidigt Google Mail för att skicka och ta emot e-post.

## Skicka ett testmail för att kontrollera om konfigurationen är framgångsrik

Ange `ops/chasquid`

Kör `direnv allow` att installera beroenden (direnv har installerats i den tidigare initieringsprocessen med en nyckel och en krok har lagts till i skalet)

spring sedan

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Innebörden av parametrarna är som följer

* användare: SMTP-användarnamn
* pass: SMTP-lösenord
* till: mottagare

Du kan skicka ett testmail.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Det rekommenderas att använda Gmail för att ta emot testmeddelanden för att kontrollera om konfigurationerna är framgångsrika.

### TLS standardkryptering

Som visas i figuren nedan finns detta lilla lås, vilket betyder att SSL-certifikatet har aktiverats.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Klicka sedan på "Visa original e-post"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Som visas i bilden nedan visar Gmails ursprungliga e-postsida DKIM, vilket betyder att DKIM-konfigurationen är framgångsrik.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Kontrollera Mottaget i rubriken på det ursprungliga e-postmeddelandet och du kan se att avsändaradressen är IPV6, vilket betyder att IPV6 också har konfigurerats framgångsrikt.
