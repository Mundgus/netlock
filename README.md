# NETLOCK próbafeladat

## Feladatleírás
**Feladat**: Hitelesítési lánc létrehozása és végfelhasználói tanúsítványok kezelése


**Cél**: A feladat célja egy hitelesítési lánc létrehozása, amely tartalmazza a legfelsőbb szintű kiadót (Root CA), valamint két láncolt kiadót. Az egyik láncolt kiadó aláírói tanúsítványokat ad ki, a másik pedig TLS tanúsítványokat.


### Feladatok:
#### Hitelesítési lánc létrehozása:
- Hozz létre egy legfelsőbb szintű kiadót (Root CA).
- Készíts két láncolt tanúsítvány kiadót, amelyek a legfelsőbb szintű kiadó alá vannak rendelve.
- Az egyik tanúsítványkiadó minősített aláíró tanúsítványokat a másik pedig TLS tanúsítványokat fog kibocsátani.


#### A végfelhasználói tanúsítványok kiadása:
- A minősített üzleti célra használható aláírói tanúsítvány érvényessége 2 év legyen.
- A TLS tanúsítványok érvényességét te határozd meg. A TLS tanúsítvány az alábbi domainekre legyen használható: *media.hu*, *alfa.media.hu*, *beta.media.hu*, *delta.media.hu*


#### Tanúsítvány visszavonás:
- A minősített aláírói tanúsítványt vonjuk vissza. A visszavonási ok nem a magánkulcs kompromittálódása miatt történjen, hanem azért mert a tanúsítványra már nincs szükség.
- Az aláírói tanúsítvány visszavonását követően frissítsd a megfelelő visszavonási listát (CRL).


#### Tanúsítványok exportálása:
-  A kiadói és végfelhasználói tanúsítványokat base64 formátumban kell exportálni.
- A végfelhasználói kulcsokat szintén base64 formátumban kell exportálni.
- A visszavonási listákat (CRL) base64 formátumban kell exportálni.


#### Indoklás a TLS tanúsítványok használhatóságáról:
- Indokold meg, miért nem teljes értékű a kiadott TLS tanúsítvány.

### Követelmények:
- A feladatot olyan eszközökkel kell végrehajtani, amelyek lehetővé teszik a tanúsítványok és kulcsok generálását, exportálását és visszavonásának kezelését.
- Az adott eszközben a műveletek végrehajtásához használt parancsok dokumentálása.
- A tanúsítványok és a visszavonási listák formátuma a szabványos X.509 formátumban legyen.


## Megoldás
### Környezet / eszközök
A feladatot Linuxon oldottam meg OpenSSL segítségével. Az OpenSSL verziója:
**OpenSSL 3.4.1 11 Feb 2025 (Library: OpenSSL 3.4.1 11 Feb 2025)**

### Feladatok
#### Hitelesítési lánc létrehozás
Root CA létrehozása:
``` Bash
openssl genpkey -algorithm RSA -out root.key -aes256
openssl req -key root.key -new -x509 -out root.crt -days 3650
```

A minősített aláírói tanúsítvány kiadó létrehozása:
``` Bash
openssl genpkey -algorithm RSA -out signer.key
openssl req -key signer.key -new -out signer.csr
openssl x509 -req -in signer.csr -CA root.crt -CAkey root.key -CAcreateserial -out signer.crt -days 1825
```

A TLS tanúsítvány kiadó létrehozása:
``` Bash
openssl genpkey -algorithm RSA -out tlsissuer.key
openssl req -key tlsissuer.key -new -out tlsissuer.csr
openssl x509 -req -in tlsissuer.csr -CA root.crt -CAkey root.key -CAcreateserial -out tlsissuer.crt -days 1825
```
A TLS tanúsítvány kiadó létrehozása ugyan az, mint a minősített aláíró tanúsítvány kiadó létrehozása. A könnyebb átláthatóság miatt ezen a ponton elkezdtem mappákba rendezni a fileokat. Ez nem kötelező lépés, de ajánlott. A root CA lejáratát 10 évre, míg az alárendelt tanúsítvány kiadókét 5 évre állítottam, de ez sem szükséges lépés.

#### A végfelhasználói tanúsítványok kiadása:
A TLS tanúsítvány létrehozása:

``` Bash
openssl genpkey -algorithm RSA -out media.key
vim media.cnf

openssl req -new -key media.key -out media.csr -config media.cnf
openssl x509 -req -in media.csr -CA ../tlsissuer.crt -CAkey ../tlsissuer.key -CAcreateserial -out media.crt -days 365
```
Itt azért hoztam létre a media.cnf filet, hogy a subdomaineket be tudjam állítani.


A minősített üzleti célra használható aláírói tanúsítvány létrehozása:
``` Bash
openssl genpkey -algorithm RSA -out business_signer.key
openssl req -key business_signer.key -new -out business_signer.csr
openssl x509 -req -in business_signer.csr -CA ../signer.crt -CAkey ../signer.key -CAcreateserial -out business_signer.crt -days 730
```

- A TLS tanúsítványok érvényességét te határozd meg. A TLS tanúsítvány az alábbi domainekre legyen használható: *media.hu*, *alfa.media.hu*, *beta.media.hu*, *delta.media.hu*


#### Tanúsítvány visszavonás:
A visszavonáshoz először létre kellett hozni egy crl_index.txt-t és egy crl_number file-t:
``` Bash
touch crl_index.txt
echo 01 > crl_number
```

Mivel az `openssl` alapértelmezetten nem ezeket a fileokat használja, így lemésoltam az `/etc/ssl/openssl.cnf` file-t a projektmappába, módosítottam és beállítottam, hogy ideiglenesen ezt a config filet használja:
``` Bash
# OpenSSl config másolás, módosítás, beállítás
cp /etc/openssl/openssl.cnf ./
vim openssl.cnf
# Itt módosítottam a [ CA_default ] részen belül a database és a crlnumber változókat, hogy az általam készített fileokra mutassanak

export OPENSSL_CONF=/home/mundgus/Projects/netlock/openssl.cnf
```

Ezt követően visszavontam a certet, majd létrehoztam a CRL filet:
``` Bash
openssl ca -revoke ./business_signer/business_signer.crt -keyfile signer.key -cert signer.crt -crl_reason cessationOfOperation
openssl ca -gencrl -keyfile signer.key -cert signer.crt -out signer.crl
```

#### Tanúsítványok exportálása:
A lépéseket követően a tanúsítványok, a kulcsok és a CRL is base64 formátumúak, így nincs szükség semmilyen átalakításra.

#### Indoklás a TLS tanúsítványok használhatóságáról:
Azért nem teljes értékű a tanúsítvány, mert a rootCA-t is mi készítettük. Ez olyan, mintha lenne egy hivatalos dokumentum, amit két embernek alá kell tanúzni, és a tanúk helyett is mi írnánk alá.

#### Egyéb megjegyzés
A megadott:
- PEM pass phrase: netlock
- challenge password: NETLOCK

Az RSA hosszán nem változtattam, vagyis 2048 bit, de a nagyobb biztonság érdekében lehet növelni ezzel a kapcsolóval: `-pkeyopt rsa_keygen_bits:4096)`
