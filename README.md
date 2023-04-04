# Cryptosizer telepítése AWS Lambda környezetbe Zappa segítségével

## Hozzávalók AWS oldalról

Régió: eu-centra-1 (Frankfurt régió)  

### VPC

Szükség van egy VPC-re, amely tartalmaz publikus és privát subneteket is. Mindegyikből típusú subnetből 3-3 darabot hozzunk létre, mert Frankfurt AWS régióban 3 db AZ van. Így mindegyik AZ-ra jut majd egy-egy subnet. Ez arra jó, hogy ha AWS szinten probléma van bármelyik AZ-val, akkor is megmarad az alkalmazás működése (kivéve ha az az AZ esik el, amiben a DB fut, annak HA-sítása túl költséges a projekthez). Ez nem kerül plussz költségbe!  
A VPC rendelkezzen Internet Gateway-al is. NAT gateway nem szükséges, mert a Lambda funckiókat a publikus subnetekből futatjuk majd. A biztonság így nagyon kis mértékben csökkenhet, de nem kell extra költséget fizetni a NAT gateway-re, ami havin szinten jelenlg kb 32 USD.  

Az AWS konzol VPN részénél a **Create VPC** lehetőséget válaszd ki.
Válaszd a következő lehetőslégetek:  

- VPC and more
- Auto-generate (pipa be), érték: cryptosizer
- IPv4 CIDR block: 10.0.0.0/16
- IPv6 CIDR block: No IPv6 CIDR block
- Tenancy: Default
- Number of Availability Zones (AZs): 3
- Customize AZs: eu-central-1a, eu-central-1b, eu-central-1c
- Number of public subnets: 3
- Number of private subnets: 3
- Customize subnets CIDR blocks: hagyd az alapértelmezett beállításokon
- NAT gateways ($): None
- VPC endpoints: S3 Gateway
- DNS options
- Enable DNS hostnames (pipa be)
- Enable DNS resolution (pipa be)

### IAM felhasználó a zappa részére

IAM név: cryptosizer-zappa  
Permissions:

- IAMFullAccess
- PowerUserAccess

Generáljunk Access key-t a felhasználónak!  

### IAM felhasználó S3 eléréshez

A statikus tartalmakat S3-on keresztül fogjuk kiszolgálni, nem a Lanmbda funkción keresztül. Így kevesebb lesz a Lambda funckió futási ideje, ezáltal a költésge is és használhatjuk az AWS CloudFront technológiát a statikus tartalmak gyorsabb kiszolgálásához extra költség nélkül.  

IAM név: cryptosizer-s3  
Permissions:

- AmazonS3FullAccess

### S3 bucket

A zappa létrehozza majd a részére szükséges bucket-et, ahova majd a Django forráskódját fogja feltölteni a Lambda funkció részére. Szükség van rendszerenként még egy bucket-re, amivel a Django statikus tartalmait (CSS, JS, satöbbi) fogjuk kiszolgálni. Ez a bucket nyilvános kell, hogy legyen.  
Az S3 szolgáltatás alatt válaszd ki a **Create bucket** lehetőséget.  
Fejlesztői környezethez:  

- Bucket name: cryptosizer-static-dev
- AWS Region: EU (Frankfurt) eu-central-1
- ACLs enabled
- Bucket owner preferred
- Block all public access (pipa ki)
- acknowledge that the current settings might result in this bucket and the objects within becoming public. (pipa be)
- Bucket Versioning: Disable
- Tags: project: cryptosizer-dev
- Encryption key type: Amazon S3 managed keys
- Bucket Key: Enable
- Object Lock: Disable

Produktív környzethez:  

- Bucket name: cryptosizer-static-prod
- AWS Region: EU (Frankfurt) eu-central-1
- ACLs enabled
- Bucket owner preferred
- Block all public access (pipa ki)
- acknowledge that the current settings might result in this bucket and the objects within becoming public. (pipa be)
- Bucket Versioning: Disable
- Tags: project: cryptosizer-prod
- Encryption key type: Amazon S3 managed keys
- Bucket Key: Enable
- Object Lock: Disable

Mindkettő bucket Permissions fül alatt a Cross-origin resource sharing (CORS) alatt vedd fel a következő beállítás:  

```json
[
    {
        "AllowedHeaders": [
            "Authorization"
        ],
        "AllowedMethods": [
            "GET"
        ],
        "AllowedOrigins": [
            "*"
        ],
        "ExposeHeaders": [],
        "MaxAgeSeconds": 3000
    }
]
```

### Elastic Cache - Redis Cluster

A Django Select2 számára szükséges egy cache megoldás. A Django támogatja a Redis-t. Ezért az AWS-ben elérhető Elastic Cache szolgáltatáson belül létrehozok egy Redis Cluster-t. Ezt fogja az alkalmazás használni. Az aszinkron feladat feldolgozáshoz szükséges Celery részére is megfelel majd a Redis.  
A létrehozást az AWS Management Console-on végzem el az Elastic Cache alatt. Arra figyelek, hogy a korábban létrehozott VPC privát subnet-jeit választom ki a Redis 7 részére. A dokumentáció írásakor a 7.0.7-es verzió az elérhető. A költséghatékonyság és SLA figyelembe vételével nem választom a cluster-be szervezett Redis megoldást. Standalone szolgáltatásként fogom használni, de az AWS automatikus HA-t biztosít ha az adott AZ-vel probléma lenne, elindítja a szolgáltatást egy másik AZ-ban.  

- Choose a cluster creation method: Easy create
- Configuration: Demo
- Cluster info, Name: cryptosizer

A Redis szolgáltatás elkészülte után, engedélyezzük a Security Group-ban, hogy a Lambda funkciókra állított Security Group-ból elfogadja a bejövő 6379/TCP kapcsolatokat. Ezt a Redis instanciára kattintva a *Network and security* fül alatt találom meg. Az ott található Security group ID-ra kattintva tudom majd beállítani a szükséges *Inbound* szabályt.

## Django lokális telepítés

A zappa miatt a jelenleg elérhető legutolsó verzióját használd a Python 3.9 szériából (3.9.16).

```bash
pyenv shell 3.9.16
python -V
    Python 3.9.16
mkdir cryptosizer-aws
cd cryptosizer-aws
python -m venv .venv
source .venv/bin/activate
```

Telepítsd a szükséges minimális Python csomagokat:  

```bash
python -m pip install django zappa
```

A dokumentum írásakor ezek a csomagok és verziók települtek:  

```txt
argcomplete==3.0.5
asgiref==3.6.0
boto3==1.26.104
botocore==1.29.104
certifi==2022.12.7
cfn-flip==1.3.0
charset-normalizer==3.1.0
click==8.1.3
Django==4.1.7
durationpy==0.5
hjson==3.1.0
idna==3.4
jmespath==1.0.1
kappa==0.6.0
MarkupSafe==2.1.2
placebo==0.9.0
python-dateutil==2.8.2
python-slugify==8.0.1
PyYAML==6.0
requests==2.28.2
s3transfer==0.6.0
six==1.16.0
sqlparse==0.4.3
text-unidecode==1.3
toml==0.10.2
tqdm==4.65.0
troposphere==4.3.2
urllib3==1.26.15
Werkzeug==2.2.3
zappa==0.56.1
```

Hozzuk létre a kiinduló Django projektet:  

```bash
django-admin startproject cryptosizer .
rehash
```

Indítsd el a Django fejlesztői kiszolgáló környzetét és nyisd meg a <http://127.0.0.1:8000/> URL-t böngészőből.  

```bash
python manage.py runserver
```

Ha betöltődik a Django kezdő oldala, akkor az alkalmazás szerver jól működik, elkezdheted az AWS Lambda kialakítást. A **python manage.py migrate** parancsot még **NE** futtasd meg, mert a felhasználók kezelése nem az alapértelmezett módon foig türténni!

## Zappa minimális Django deployment adatbázis támogatás nélkül

AWS Lambda környezetben a Django alapértelmezett SQLite3 megoldása nem támogatott, mivel a lokális fájlrendszer alapértelmezetten nem perzisztens. Ezért a *cryptosizer/settings.py* állományban kommentezd ki a DATABASE szekciót, hogy így nézzen ki.  

```python
# DATABASES = {
#     "default": {
#         "ENGINE": "django.db.backends.sqlite3",
#         "NAME": BASE_DIR / "db.sqlite3",
#     }
# }
```

### SECRET_KEY generálása

```bash
python manage.py shell
from django.core.management.utils import get_random_secret_key
print(get_random_secret_key())
x#t12qe*qfq31u0kpot$!m7mnu0gx)#=c6q+obyi&(3h3ct7h2
```

Az így kapott karakterhalmazt használd fel a SECRET_KEY paraméter értékének a *cryptosizer/settings.py* állományban.

### ALLOWED_HOST paraméter beállítása

Egyelőre ezt állítsd be. Majd a zappa első deploy után fogod megkapni az AWS endpoint URL-t, amivel ki kell egészíteni.

```python
ALLOWED_HOSTS = [
    "127.0.0.1",
    "localhost"
]
```

## AWS Lambda dev környezet kitelepítése

Korábbi fejezetben telepítetted már a *zappa* Python csomagot. Végezd el az alap inicializálását. Lépj be a projekt fő könyvtárába, ahol a Djano *manage.py* állomány is található. Onnan végezd el a feadatot. A parancs interaktív módon feltesz néhány kérdést, azokra add meg a helyes választ. A saját válaszaimat félkövéren jeölöm.  
Első lépésként vedd fel a zappa profilt az AWS credentials-ba a *~/.aws/credentials* állományba. Itt használd a korábban létrehozott *cryptosizer-zappa* IAM felhasználó access key-t és secret access key-t.  

```txt
[default]
aws_access_key_id = 
aws_secret_access_key = 

[zappa]
aws_access_key_id = XXXXXXXXXXXX
aws_secret_access_key = YYYYYYYYYYYYYY
```

Indítsd el a zappa inicializálását.  

```bash
zappa init
```

Your Zappa configuration can support multiple production stages, like 'dev', 'staging', and 'production'.
What do you want to call this environment (default 'dev'): **dev**  

AWS Lambda and API Gateway are only available in certain regions. Let's check to make sure you have a profile set up in one that will work.
We found the following profiles: default, zappa, and admin-cli. Which would you like us to use? (default 'default'): **zappa**  

Your Zappa deployments will need to be uploaded to a private S3 bucket.
If you don't have a bucket yet, we'll create one for you too.
What do you want to call your bucket? (default 'zappa-g2qt1m1b4'): **cryptosizer-dev-g2qt1m1b4**  

It looks like this is a Django application!
What is the module path to your projects's Django settings?
We discovered: cryptosizer.settings
Where are your project's settings? (default 'cryptosizer.settings'): **cryptosizer.settings**  

You can optionally deploy to all available regions in order to provide fast global service.
If you are using Zappa for the first time, you probably don't want to do this!
Would you like to deploy this application globally? (default 'n') [y/n/(p)rimary]: **n**  

Okay, here's your zappa_settings.json:

{
    "dev": {
        "django_settings": "cryptosizer.settings",
        "profile_name": "zappa",
        "project_name": "cryptosizer-aws",
        "runtime": "python3.9",
        "s3_bucket": "cryptosizer-dev-g2qt1m1b4"
    }
}

Does this look okay? (default 'y') [y/n]: **y**  

```txt
Done! Now you can deploy your Zappa application by executing:

    $ zappa deploy dev

After that, you can update your application code with:

    $ zappa update dev

To learn more, check out our project page on GitHub here: https://github.com/Zappa/Zappa
and stop by our Slack channel here: https://zappateam.slack.com

Enjoy!,
 ~ Team Zappa!
```

Ezzel el is készült a zappa alap inicializálása és létrejött a *zappa_settings.json* konfigurációs állomány. Ez még nem elgéséges az első kitelepítéshez, mivel a VPC és subnet információkat is szeretném megadni. Korábbi fejezetben létrehozott Subnet (csak a publikusakat!) és Security Group (az alapértelmezettet) ARN/ID-kat gyűjtsd össze és azokat cseréld le a *zappa_settings.json* állományban. Arra figyelj oda, hogy ha több VPC-d van, akkor több default Security Group-od lehet. Minden VPC-hez tartozik egy. A megfelelő VPC SG ID-ját másold be. Ugyanez igaz a Subnet ID-kra is.  

```json
{
    "dev": {
        "django_settings": "cryptosizer.settings",
        "profile_name": "zappa",
        "project_name": "cryptosizer-aws",
        "runtime": "python3.9",
        "s3_bucket": "cryptosizer-dev-g2qt1m1b4",
        "timeout_seconds": 10,
        "route53_enabled": true,
        "aws_region": "eu-central-1",
        "vpc_config": {
            "SubnetIds": [ "subnet-05a152d5156fc0c74", "subnet-019fa9775e40a147f", "subnet-08fc45dbdbe8b9e6a" ],
            "SecurityGroupIds": [ "sg-032f1dc5939ad7a72" ]
        }
    }
}
```

### Első AWS Lambda kitelepítés

A korábbi lépések elvégzésével már kitelepíthetjük az első Lambda funkciót és tesztelhetjük majd az elérését. A zappa parancs (mivel rendelkezik megfelelő AWS jogosultásgokkal) létre fogja hozni a szükséges erőforrásokat és IAM felhazsnálókat. Majd feltölti a forráskód összecsomagolt verzióját az AWS Lambda funkcióba és megvárja, amíg a Lambda funkció elkészül. Ez pár percig is eltarthat.  

```bash
zappa deploy dev
Calling deploy for stage dev..
Creating cryptosizer-aws-dev-ZappaLambdaExecutionRole IAM Role..
Creating zappa-permissions policy on cryptosizer-aws-dev-ZappaLambdaExecutionRole IAM Role.
Downloading and installing dependencies..
 - markupsafe==2.1.2: Downloading
100%|██████████████████████████████████████████████████████████████████████████████████| 25.5k/25.5k [00:00<00:00, 12.2MB/s]
 - charset-normalizer==3.1.0: Using locally cached manylinux wheel
Packaging project as zip.
Uploading cryptosizer-aws-dev-1680514924.zip (15.4MiB)..
100%|██████████████████████████████████████████████████████████████████████████████████| 16.1M/16.1M [00:03<00:00, 4.88MB/s]
Waiting for lambda function [cryptosizer-aws-dev] to become active...
Scheduling..
Scheduled cryptosizer-aws-dev-zappa-keep-warm-handler.keep_warm_callback with expression rate(4 minutes)!
Uploading cryptosizer-aws-dev-template-1680515059.json (1.6KiB)..
100%|██████████████████████████████████████████████████████████████████████████████████| 1.66k/1.66k [00:00<00:00, 17.2kB/s]
Waiting for stack cryptosizer-aws-dev to create (this can take a bit)..
 75%|██████████████████████████████████████████████████████████████████                      | 3/4 [00:12<00:04,  4.18s/res]
Deploying API Gateway..
Waiting for lambda function [cryptosizer-aws-dev] to be updated...
Deployment complete!: https://ykrmsz9h46.execute-api.eu-central-1.amazonaws.com/dev
```

Ha minden jól ment, akkor a visszakapott URL-ből az FQDN részt tedd be a *cryptosizer/settings.py* állomány ALLOWED_HOST paraméterei közé. E nélkül hibaüzenetet fog adni a Django alkalmazás. Nekem így fest ez után a paraméter.  

```python
ALLOWED_HOSTS = [
    "127.0.0.1",
    "localhost",
    "ykrmsz9h46.execute-api.eu-central-1.amazonaws.com"
]
```

Ez után annyi a teendő, hogy frissítjük a forráskódot a meglévő Lambda funkcióban a következő paranccsal. Ez a parancs lesz a megfelelő mindin alkalommal, mikor a forráskódon változtatsz és azt kiakarod telepíteni.  

```bash
zappa update dev
Calling update for stage dev..
Downloading and installing dependencies..
 - markupsafe==2.1.2: Downloading
100%|██████████████████████████████████████████████████████████████████████████████████| 25.5k/25.5k [00:00<00:00, 16.9MB/s]
 - charset-normalizer==3.1.0: Using locally cached manylinux wheel
Packaging project as zip.
Uploading cryptosizer-aws-dev-1680515319.zip (15.4MiB)..
100%|██████████████████████████████████████████████████████████████████████████████████| 16.1M/16.1M [00:03<00:00, 4.97MB/s]
Updating Lambda function code..
Waiting for lambda function [cryptosizer-aws-dev] to be updated...
Updating Lambda function configuration..
Uploading cryptosizer-aws-dev-template-1680515334.json (1.6KiB)..
100%|██████████████████████████████████████████████████████████████████████████████████| 1.66k/1.66k [00:00<00:00, 19.7kB/s]
Deploying API Gateway..
Scheduling..
Unscheduled cryptosizer-aws-dev-zappa-keep-warm-handler.keep_warm_callback.
Scheduled cryptosizer-aws-dev-zappa-keep-warm-handler.keep_warm_callback with expression rate(4 minutes)!
Waiting for lambda function [cryptosizer-aws-dev] to be updated...
Your updated Zappa deployment is live!: https://ykrmsz9h46.execute-api.eu-central-1.amazonaws.com/dev
```

Ez után már böngészőből megnyithatóvá válik az URL. Mivel nincs még egyetlen egy alkalmazásunk és View-nk sem, ezért Page not found (404) teljesen normális. Egyedül a */admin* URL lesz még elérhető. Nyisd is meg!  

### Statikus állományok kiszolgálása S3 bucket-ből

Miután megnyitottad a */admin* aloldalt, láthatod, hogy a CSS leírók mint ha nem töltődtek volna be. Ez ebben a pillanatban teljesen rendben van. Gondoskodni kell a statikus állományok, így a CSS stílusleírók kiszolgálásáról is. Én S3 bucket eléréssel oldom meg. Ehhez telepíteni kell egy új Píthon csomagot.  

```bash
python -m pip install django-s3-storage
```

Vegyük fel a *cryptosizer/settings.py* állományban az **INSTALLED_APPS** közé.  

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django_s3_storage", # Uj
]
```

Majd egészítsük ki *cryptosizer/settings.py* állományt a szükséges S3 és AWS paraméterekkel.  

```python
# Django S3 Storage
YOUR_S3_BUCKET = 'cryptosizer-static-dev'
DEFAULT_FILE_STORAGE = "django_s3_storage.storage.S3Storage"
STATICFILES_STORAGE = "django_s3_storage.storage.StaticS3Storage"
AWS_S3_CUSTOM_DOMAIN = '%s.s3.amazonaws.com' % YOUR_S3_BUCKET
AWS_S3_BUCKET_NAME_STATIC = YOUR_S3_BUCKET
STATIC_URL = 'https://%s/' % AWS_S3_CUSTOM_DOMAIN
AWS_S3_BUCKET_AUTH_STATIC = False
AWS_S3_USE_THREADS = True
AWS_S3_MAX_POOL_CONNECTIONS = 10
AWS_S3_CONNECT_TIMEOUT = 30
AWS_REGION = "eu-central-1"
AWS_ACCESS_KEY_ID = "XXXXXXXXXXXXX"
AWS_SECRET_ACCESS_KEY = "YYYYYYYYYYYYY"
```

Futassd meg a *collectstatic* Django manage parancsot, hogy a statikus állományok feltöltésre kerüljenek a megfelelő S3 bucket-ba. Ezt ellenőrizd is majd le az AWS Console felületén.  

```bash
python manage.py collectstatic           

You have requested to collect static files at the destination
location as specified in your settings.

This will overwrite existing files!
Are you sure you want to do this?

Type 'yes' to continue, or 'no' to cancel: yes

130 static files copied.
```

Mivel változtattunk a *settings.py* állományon, frissíteni kell a Lambda funckió forráskódját is. 
Végezd el az imsert paranccsal.  

```bash
zappa update dev
```

Ezek után nyisd meg a */admin* URL-t és ha minden jól ment, az oldal már sokkal szebben fog megjelenni, mivel a CSS-eket eléri a böngésződ S3-on keresztül. Belépni még nem tudunk, mivel nem hoztam létre superuser fiókot és még az adatbázissal sem kapcsoltam össze. A következő lépésekben ezt fogom megtenni.  

### Postgre RDS szolgáltatás létrehozása AWS-ben

Az adatbázis erőforrást RDS szolgáltatásként hozom létre és érem majd el. Így az adatbázis üzemeltetése is az AWS feladata lesz. Ehhez létrehozok egy új RDS szolgáltatást.  

- Choose a database creation method: Standard create
- Engine type: PostgreSQL
- Engine Version: PostgreSQL 14.6-R1
- Templates: Free tier
- DB instance identifier: cryptosizerdev
- Master username: postgres (más is lehet!)
- Manage master credentials in AWS Secrets Manager (pipa ki)
- Auto generate a password (pipa be)
- DB instance class: db.t4g.micro
- Storage type: General Purpose SSD (gp2)
- Allocated storage: 20 GiB
- Enable storage autoscaling (pipa ki)
- Compute resource: Don’t connect to an EC2 compute resource
- Network type: IPv4
- Virtual private cloud (VPC): cryptosizer-vpc
- DB subnet group: Create new DB Subnet Group
- Public access: No
- VPC security group (firewall): Create new
- New VPC security group name: cryptosizer-db-dev-sg
- Availability Zone: No preference
- Certificate authority - optional: rds-ca-2019 (default)
- Database port: 5432
- Database authentication options: Password authentication
- Additional configuration -> Initial database name: cryptosizerdev
- Enable automated backups (pipa ki)
- Enable encryption (pipa be)
- AWS KMS key: (default)
- Log exports: PostgreSQL Log (pipa be), Upgrade log (pipa be)
- Enable auto minor version upgrade: pipa be
- Choose window

Megvárom, amíg az RDS szolgáltatás elkészül (pár perc), majd a szolgáltatást kiválasztva megkeresem az Endpoint címet, ami szükséges lesz majd a Django adatbázis konfigurációhoz.

### Adatbázis konfiguráció létrehozása Django alkalmazásban

A létrehozott adatbázis paramétereit a korábban kikommentezett rész helyére felveszem a *cryptosizer/settings.py* állományba.  

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "cryptosizerdev",
        "USER": "postgres",
        "PASSWORD": "XXXXXXXXXXXX",
        "HOST": "cryptosizerdev.c7k7twyplugp.eu-central-1.rds.amazonaws.com",
        "PORT": "5432",
    }
}
```

Ahhoz tesztelni tudjuk, első lépésként feltöltöm az új forráskódot és telepítem a *psycopg2-binary* Python csomagot lokálisan.

```bash
python -m pip install psycopg2-binary
zappa update dev
```

Miután sikeresen feltöltődött az új forráskód, letesztelem a Django PostgreSQL elérését.  

```bash
zappa manage dev inspectdb
[START] RequestId: 8bc0262c-2da8-428a-ab5a-86a5f12bcfcc Version: $LATEST
[DEBUG] 2023-04-03T10:40:38.341Z 8bc0262c-2da8-428a-ab5a-86a5f12bcfcc Zappa Event: {'manage': 'inspectdb'}
2023-04-03T10:41:08.374Z 8bc0262c-2da8-428a-ab5a-86a5f12bcfcc Task timed out after 30.03 seconds
[END] RequestId: 8bc0262c-2da8-428a-ab5a-86a5f12bcfcc
[REPORT] RequestId: 8bc0262c-2da8-428a-ab5a-86a5f12bcfcc
Duration: 30033.67 ms
Billed Duration: 30000 ms
Memory Size: 512 MB
Max Memory Used: 98 MB
```

Látszik, hogy két probléma is van. Az első, hogy nem engedtem meg a PostgreSQL Security Group-ban, hogy a Lambda Security Group-ból beengedje a TCP/5432 kapcsolatokat. A másik a túl magas Timeout érték. Ami alaból 30 másodperc, azaz 30000 ms. A Lambda funkció elszámolása idő alapú (millisec). Ezért csökkentem a temeout értéket 10 sec-re. a *zappa_settings.json* állományba.  

```json
{
    "dev": {
        "django_settings": "cryptosizer.settings",
        "profile_name": "zappa",
        "project_name": "cryptosizer-aws",
        "runtime": "python3.9",
        "s3_bucket": "cryptosizer-dev-g2qt1m1b4",
        "timeout_seconds": 10,
        "route53_enabled": true,
        "vpc_config": {
            "SubnetIds": [ "subnet-XXXXXX", "subnet-XXXXXXX", "subnet-XXXXXXX" ],
            "SecurityGroupIds": [ "sg-XXXXXX" ]
        }
    }
}
```

Majd a **zappa update dev** paranccsal frissítem az infrastruktúrát és közben hozzáadom a Security Group-ban a szükséges Inbound szabályt az adatbázis SG-jében.  
Ez után újra tesztelem a Django DB elérését.  

```bash
zappa manage dev inspectdb
[START] RequestId: 871cb4eb-8938-4971-8019-0b33e5803c51 Version: $LATEST
[DEBUG] 2023-04-03T10:48:25.887Z 871cb4eb-8938-4971-8019-0b33e5803c51 Zappa Event: {'manage': 'inspectdb'}
# This is an auto-generated Django model module.
# You'll have to do the following manually to clean this up:
#   * Rearrange models' order
#   * Make sure each model has one field with primary_key=True
#   * Make sure each ForeignKey and OneToOneField has `on_delete` set to the desired behavior
#   * Remove `managed = False` lines if you wish to allow Django to create, modify, and delete the table
# Feel free to rename the models, but don't rename db_table values or field names.
from django.db import models
[END] RequestId: 871cb4eb-8938-4971-8019-0b33e5803c51
[REPORT] RequestId: 871cb4eb-8938-4971-8019-0b33e5803c51
Duration: 170.57 ms
Billed Duration: 171 ms
Memory Size: 512 MB
Max Memory Used: 103 MB
```

A kapcsolat ezúttal sikeres volt. Így a Privát subnet-ban lévő PostgreSQL adatbázishoz tud kapcsolódni a Publikus Subnet-ben futó Lambda funkció, azaz a Django.

## Django környezet kialakítása

Az alapértelmezett Django telepítésen váltztatok pár helyen:  

- Argon2 jelszó hash használata (x86_64)
- Időzóna beállítása
- Alkalmazás szintű *template* használat
- Custom User modell használata és AUTH_USER_MODEL paraméter beállítása
- Superuser létrehozása

### Argon2 jelszó hash használata

<https://docs.djangoproject.com/en/4.1/topics/auth/passwords/#using-argon2-with-django>  
Sajnos arm64 platformon fejlesztőknek nem lehet bekapcsolni az Argon2 has-t, mert az AWS Lambda jelenleg ezt nem támogatja. A zappas nem támogatja a cikk írásakor az architektúra definiálását. A jelszó hash módosítását utólag is meglehet majd tenni, miután támogatottá válik.  

A *cryptosizer/settings.py* állományban definiálom az alapértelmezett és további jelszó hash algoritmusokat.  

```python
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.Argon2PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',
    'django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher',
    'django.contrib.auth.hashers.BCryptSHA256PasswordHasher',
    'django.contrib.auth.hashers.ScryptPasswordHasher',
]
```

Ez után telepítem a szükséges Python csomagokat.  

```bash
python -m pip install "django[argon2]"
```

### Időzóna beállítása

A *cryptosizer/settings.py* állományban definiálom az alapértelmezett időzóna beállítása.  

```python
TIME_ZONE = "Europe/Budapest"
```

### Alkalmazás szintű *template* használat

Módosítom a  *cryptosizer/settings.py* állományt.  

```python
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / 'templates'], # Új
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]
```

### Custom User modell használata

Első lépésként létrehozom az *account* alkalmazást a Django projektben. Ez a alkalmazás lesz felelős az egyedi felhasználó model kezeléséért.

```bash
python manage.py startapp accounts
```

Az alkalmazást felveszem a *cryptosizer/settings.py* állomány INSTALLED_APPS paraméterbe.  

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django_s3_storage",
    "account.apps.AccountConfig", # Uj
]
```

Módosítom a *accounts/models.py* állományt. Itt egy egyedi, Webhook mezőt tudunk hozzárendelni a felhasználókhoz.  

```python
import uuid
from django.db import models
from django.contrib.auth.models import AbstractUser


class Webhook(models.Model):
    id = models.UUIDField(
        primary_key=True,
        default=uuid.uuid4,
        editable=False,
    )
    created = models.DateTimeField(auto_now_add=True, editable=False)
    last_modified = models.DateTimeField(auto_now=True, editable=False)
    name = models.CharField(verbose_name='Webhooks name', max_length=50, null=False, blank=False)
    url = models.URLField(max_length=255, null=False, blank=False, unique=True, default='https://change.me')

    def __str__(self):
        return self.name

    class Meta:
        ordering = ['name']


class CustomUser(AbstractUser):

    webhooks = models.ManyToManyField(Webhook, blank=True)

    def __str__(self):
        return self.username
```

Módosítom az *accounts/admin.py* állományt.  

```python
from django.contrib import admin
from django.contrib.auth import get_user_model
from django.contrib.auth.admin import UserAdmin

from .forms import CustomUserCreationForm, CustomUserChangeForm
from .models import Webhook

CustomUser = get_user_model()


class CustomUserAdmin(UserAdmin):
    add_form = CustomUserCreationForm
    form = CustomUserChangeForm
    model = CustomUser
    list_display = [
        "username",
        "email",
        "is_superuser",
        "is_staff",
        "is_active",
    ]
    fieldsets = UserAdmin.fieldsets + ((None, {"fields": ("webhooks",)}),)
    add_fieldsets = UserAdmin.add_fieldsets + ((None, {"fields": ("webhooks",)}),)


class WebhookAdmin(admin.ModelAdmin):
    list_display = (
        'name',
    )


admin.site.register(Webhook, WebhookAdmin)
admin.site.register(CustomUser, CustomUserAdmin)
admin.site.site_header = 'Cryptosizer Admin'
```

Létrehozom a *accounts/forms.py* állományt.  

```python
from django import forms
from django.contrib.auth.forms import UserCreationForm, UserChangeForm, AuthenticationForm
from .models import CustomUser


class CustomUserCreationForm(UserCreationForm):

    class Meta:
        model = CustomUser
        fields = ("username", "email")


class CustomUserChangeForm(UserChangeForm):

    class Meta:
        model = CustomUser
        fields = ("username", "email")
```

Módosítom a *cryptosizer/settings.py* állományt.  

```python
AUTH_USER_MODEL = "accounts.CustomUser"
```

Feltöltöm az új forráskódot.  

```bash
zappa update dev
...
Your updated Zappa deployment is live!: https://ykrmsz9h46.execute-api.eu-central-1.amazonaws.com/dev
```

Ideiglenesen visszaállítom a lokális forráskódban az SQLite3 adatbázis használatát. Ugyanis a Lambda funkcióban futó Djangó forráskód csak olvasható fájlrendszeren fut (kivéve a /tmp). Így a *makemigrations* parancs nem tudná kiírni a modelek létrehozásának a leíró állományait. Ezt minden esetben meg kell tennem, mikor model-t módosítunk, hozok létre vagy törlö.  

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}
```

Majd lefuttatom a *makemigrations* parancsot.  

```bash
python manage.py makemigrations
Migrations for 'accounts':
  accounts/migrations/0001_initial.py
    - Create model Webhook
    - Create model CustomUser
```

Létrehozza a parancs a szükséges állományokat. Ez után visszaállítom az adatbázis kapcsolatot az AWS PostgreSQL instanciára. Ez után ismét feltöltöm a forráskódot. Az előző *zappa update dev* parancs ki is lett volna hagyható ebben az esetben.

```bash
zappa update dev
...
Your updated Zappa deployment is live!: https://ykrmsz9h46.execute-api.eu-central-1.amazonaws.com/dev
```

A forráskód frissítése után lefuttatom a modelek létrehozásáért felelős parancsot.  

```bash
zappa manage dev migrate
[START] RequestId: 9879a6aa-6254-4d84-aa27-6d5d656de48f Version: $LATEST
[DEBUG] 2023-04-03T12:58:22.138Z 9879a6aa-6254-4d84-aa27-6d5d656de48f Zappa Event: {'manage': 'migrate'}
Operations to perform:
Apply all migrations: accounts, admin, auth, contenttypes, sessions
Running migrations:
Applying contenttypes.0001_initial... OK
Applying contenttypes.0002_remove_content_type_name... OK
Applying auth.0001_initial... OK
Applying auth.0002_alter_permission_name_max_length... OK
Applying auth.0003_alter_user_email_max_length... OK
Applying auth.0004_alter_user_username_opts... OK
Applying auth.0005_alter_user_last_login_null... OK
Applying auth.0006_require_contenttypes_0002... OK
Applying auth.0007_alter_validators_add_error_messages... OK
Applying auth.0008_alter_user_username_max_length... OK
Applying auth.0009_alter_user_last_name_max_length... OK
Applying auth.0010_alter_group_name_max_length... OK
Applying auth.0011_update_proxy_permissions... OK
Applying auth.0012_alter_user_first_name_max_length... OK
Applying accounts.0001_initial... OK
Applying admin.0001_initial... OK
Applying admin.0002_logentry_remove_auto_add... OK
Applying admin.0003_logentry_add_action_flag_choices... OK
Applying sessions.0001_initial... OK
[END] RequestId: 9879a6aa-6254-4d84-aa27-6d5d656de48f
[REPORT] RequestId: 9879a6aa-6254-4d84-aa27-6d5d656de48f
Duration: 1124.72 ms
Billed Duration: 1125 ms
Memory Size: 512 MB
Max Memory Used: 105 MB
```

A kimenetből látszik, hogy a migráció rendben lefutott, létrejöttek a megfelelő adatbázis objektumok. Ezért az AWS **1125 ms** futási időt számolt fel.

### Superuser létrehozása

Még egy lépés kell ahhoz, hogy teljes értékű Django keretrendszert kapjunk. Létre kell hoznom egy superuser-t. A Lambda funkcióval nem lehet interactive módon létrehozni ezt a felhasználót. nem támogatja az interactive megoldásokat. A zappa eszköznek viszont van egy olyan lehetősége, hogy a Lambda funkción Python script-et futtathatok (ezt megtehetem az AWS Management Console segítségével is). Tetszőleges értékeket megadhatsz, nem szükséges, hogy *admin* legyen a felhasználó neve, sőt...!

```bash
zappa invoke --raw dev "from django.contrib.auth import get_user_model; User = get_user_model(); User.objects.create_superuser('admin', 'admin@cryptosizer.app', 'SzuperJelszoAdminnak')"[START] RequestId: d64d0bc5-13d3-417a-a22c-c44c904653b7 Version: $LATEST
[DEBUG] 2023-04-03T13:57:36.802Z d64d0bc5-13d3-417a-a22c-c44c904653b7 Zappa Event: {'raw_command': "from django.contrib.auth import get_user_model; User = get_user_model(); User.objects.create_superuser('admin', 'admin@cryptosizer.app', 'SzuperJelszoAdminnak')"}
[END] RequestId: d64d0bc5-13d3-417a-a22c-c44c904653b7
[REPORT] RequestId: d64d0bc5-13d3-417a-a22c-c44c904653b7
Duration: 987.50 ms
Billed Duration: 988 ms
Memory Size: 512 MB
Max Memory Used: 105 MB
```

### Külső Django alkalmazások telepítése

A következő Django alkalmazásokat fogom telepíteni:

- Django Select2
- Crispy Forms
- Crispy Bootstrap5
- Bootstrap Datepicker Plus
- Debug Toolbar
- Django Filters
- Axes

#### Django Select 2 telepítése, konfigurálása

Alkalmazás dokumentáció: <https://django-select2.readthedocs.io/en/latest/>  
Korábbi fejezetben létrehoztam az AWS-ben egy Redis szolgáltatást. A szolgáltatás *Configuration endpoint* értéke szükséges számomra, hogy bekonfiguráljam a Django és a Select2 alá a Redis cache-t.  
Ez az érték az én clusteremben: cryptosizer.68idna.clustercfg.euc1.cache.amazonaws.com:6379  
A te értéked minden bizonnyal eltérő lesz.  
Következő lépésként telepítem a szükséges Python csomagot és függőségeit és bővítem a *requirements.txt* állományt.  

```bash
python3 -m pip install django-select2 django-redis
```

Hozzáadom a Select2 alkalmazást a *cryptosizer/settings.py* állományban az INSTALLED_APPS-hoz.  

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django_s3_storage",
    "django_select2", # Új
    "accounts.apps.AccountsConfig",
]
```

requirements.txt:  

```ini
django-select2==8.1.1
django-redis==5.2.0
```

Hozzáadom a *django_select*-et a *cryptosizer/urls.py* állományhoz.  

```python
from django.contrib import admin
from django.urls import path
from django.urls import include # Új

urlpatterns = [
    path("admin/", admin.site.urls),
    path("select2/", include("django_select2.urls")), # Új
]
```

Definiálom a *CACHES* paramétert a *cryptosizer/settings.py* állományba. Django cache dokumentációja: <https://docs.djangoproject.com/en/4.1/topics/cache/#redis>  
A *LOCATION* paraméter értékének adom meg a korábban kigyűjtött *cryptosizer.68idna.clustercfg.euc1.cache.amazonaws.com:6379* értéket.  

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://cryptosizer.68idna.clustercfg.euc1.cache.amazonaws.com:6379',
    },
    "select2": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://cryptosizer.68idna.clustercfg.euc1.cache.amazonaws.com:6379/2",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    }
}

SELECT2_CACHE_BACKEND = "select2"
```

Frissítem a forráskódot és letesztelem az admin felületre történő belépést.  

```bash
zappa update dev
```

#### Crispy Forms és Crispy Bootstrap5 telepítése

Alkalmazás dokumentáció: <https://django-crispy-forms.readthedocs.io/en/latest/>  

Telepítem a szükséges Python csomagot és függőségeit és bővítem a *requirements.txt* állományt.  

```bash
python3 -m pip install django-crispy-forms crispy-bootstrap5
```

```ini
django-crispy-forms==2.0
crispy-bootstrap5==0.7
```

Hozzáadom a Crispy forms alkalmazást a *cryptosizer/settings.py* állományban az INSTALLED_APPS-hoz.  

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django_s3_storage",
    "django_select2",
    "crispy_forms", # Új
    "crispy_bootstrap5", # Új
    "accounts.apps.AccountsConfig",
]
```

Hozzáadom az alkalmazások konfigurációját a *cryptosizer/settings.py* állományhoz.  

```python
# Crispy Bootsrap 5
CRISPY_ALLOWED_TEMPLATE_PACKS = "bootstrap5"
CRISPY_TEMPLATE_PACK = "bootstrap5"
```

Frissítem a forráskódot a Lambda funkcióban.  

```bash
zappa update dev
```

#### Bootstrap Datepicker Plus telepítése

Az alkalmazás dokumentációja: <https://django-bootstrap-datepicker-plus.readthedocs.io/en/latest/index.html>  

Telepítem a szükséges Python csomagokat.  

```bash
python3 -m pip install django-bootstrap-datepicker-plus
```

Frissítem a *requirements.txt* állományt.  

```ini
django-bootstrap-datepicker-plus==5.0.3
```

Hozzáadom az alkalmazást a *cryptosizer/settings.py* állományban az INSTALLED_APPS-hoz.  

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django_s3_storage",
    "django_select2",
    "crispy_forms",
    "crispy_bootstrap5",
    "bootstrap_datepicker_plus", # Új
    "accounts.apps.AccountsConfig",
]
```

Definiáloma a *cryptosizer/settings.py* állományban a személyre szabott beállításokat.  

```python
BOOTSTRAP_DATEPICKER_PLUS = {
    "options": {
        "showClose": True,
        "showClear": True,
        "showTodayButton": True,
        "allowInputToggle": True,
    },
    "addon_icon_classes": {
        "month": "bi-calendar-month",
    },
}
```

Az alkalmazásnak szüksége van JQuery-re is. Annak telepítése egy későbbi fejezetben lesz leírva.  
Frissítem a forráskódot a Lambda funkcióban.  

```bash
zappa update dev
```

#### Debug Toolbar telepítése

Az alkalmazás dokuemntációja: <https://django-debug-toolbar.readthedocs.io/en/latest/>  
Telepítem a szükséges Python csomagot.  

```bash
python3 -m pip install django-debug-toolbar
```

Frissítem a *requirements.txt* állományt.  

```ini
django-debug-toolbar==4.0.0
```

Ellenőrzöm az előfeltétel meglétét a *cryptosizer/settings.py* állományban az INSTALLED_APPS-ban.  

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles", # Erre szüksége van a csomagnak!
    "django_s3_storage",
    "django_select2",
    "crispy_forms",
    "crispy_bootstrap5",
    "bootstrap_datepicker_plus", # Új
    "accounts.apps.AccountsConfig",
]
```

A TEMPLATES paraméterbenn az *APP_DIRS* értéke legyen *True*.  

```python
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / 'templates'],
        "APP_DIRS": True, # Ezt a paramétert kell ellenőrizni!
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]
```

Hozzáadom az alkalmazást a *cryptosizer/settings.py* állományban az INSTALLED_APPS-hoz.  

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django_s3_storage",
    "django_select2",
    "crispy_forms",
    "crispy_bootstrap5",
    "bootstrap_datepicker_plus",
    "debug_toolbar", # Új
    "accounts.apps.AccountsConfig",
]
```

Hozzáadom a *cryptosizer/urls.py* állományban a szükséges URL-t.  

```python
urlpatterns = [
    path("admin/", admin.site.urls),
    path("select2/", include("django_select2.urls")),
    path("__debug__/", include("debug_toolbar.urls")),
]
```

Hozzáadom a *cryptosizer/settings.py* állományban a szükséges MIDDLEWARE beállítást. Fontos a MIDDLEWARE értékek sorrendje!  

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "debug_toolbar.middleware.DebugToolbarMiddleware", # Új
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

Beállítom a szükséges INTERNAL_IPS paramétert a *cryptosizer/settings.py* állományban.  

```python
# Debug Toolbar
INTERNAL_IPS = [
    "127.0.0.1",
]
```

#### Django Filters telepítése

Az alkalmazás dokumentációja: <https://django-filter.readthedocs.io/en/stable/>  
Telepítem a szükséges Python csomagot.  

```bash
python3 -m pip install django-filter
```

Frissítem a *requirements.txt* állományt.  

```ini
django-filter==23.1
```

Hozzáadom az alkalmazást a *cryptosizer/settings.py* állományban az INSTALLED_APPS-hoz.  

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django_s3_storage",
    "django_select2",
    "crispy_forms",
    "crispy_bootstrap5",
    "bootstrap_datepicker_plus",
    "debug_toolbar",
    "django_filters", # Új
    "accounts.apps.AccountsConfig",
]
```

#### Axes telepítése

Az alkalmazás dokumentációja: <https://django-axes.readthedocs.io/en/latest/index.html>  
Telepítem a szükséges Python csomagot.  

```bash
python3 -m pip install django-axes
```

Frissítem a *requirements.txt* állományt.  

```ini
django-axes==5.41.0
```

Hozzáadom az alkalmazást a *cryptosizer/settings.py* állományban az INSTALLED_APPS-hoz.  

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "django_s3_storage",
    "django_select2",
    "crispy_forms",
    "crispy_bootstrap5",
    "bootstrap_datepicker_plus",
    "debug_toolbar",
    "django_filters",
    "axes",
    "accounts.apps.AccountsConfig",
]
```

A *cryptosizer/settings.py* állományban definiálom a szükséges *AUTHENTICATION_BACKENDS* paramétert.  

```python
AUTHENTICATION_BACKENDS = [
    # AxesStandaloneBackend should be the first backend in the AUTHENTICATION_BACKENDS list.
    'axes.backends.AxesStandaloneBackend',

    # Django ModelBackend is the default authentication backend.
    'django.contrib.auth.backends.ModelBackend',
]
```

A *cryptosizer/settings.py* állományban kibővítem a *MIDDLEWARE* paramétert.  

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "debug_toolbar.middleware.DebugToolbarMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    "axes.middleware.AxesMiddleware", # Új
]
```

A *cryptosizer/settings.py* állományban definiálom a szeméylre szabott beállításokat.  

```python
AXES_FAILURE_LIMIT = 5
AXES_LOCK_OUT_BY_COMBINATION_USER_AND_IP = True
AXES_NEVER_LOCKOUT_WHITELIST = True
AXES_IP_WHITELIST = [ "78.131.55.214", ]
AXES_RESET_ON_SUCCESS = True
AXES_PROXY_COUNT = None
AXES_META_PRECEDENCE_ORDER = [
    "HTTP_X_FORWARDED_FOR",
    "REMOTE_ADDR",
]
AXES_COOLOFF_TIME = 0.16  # 10 perc utan kilockol
```

Frissítem a forráskódot, majd az adatbázis táblákat.  

```bash
zappa update dev
zappa manage dev migrate
```
