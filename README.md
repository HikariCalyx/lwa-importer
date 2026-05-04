# lwa-importer

This is the source code of *Foreigner Character Importer for Lotte World Adventure*.

## API Endpoint

The endpoint will be closed after June 15. The original API endpoint is `LWA/v1/GetAvatarCode`.

Method: POST
Content-Type: application/json

Body format:

```json
{
    "ign": "Character name",
    "region": "internal name of game region"
}
```

Acceptable region names: InkwellMS_NA, InkwellMS_EU, ChangseopMS, ZipanguMS, TerryMS_Island, TerryMS_Peninsula, ~~TerryMS_Continent~~

Valid response body example:

```json
{
    "avatarCode": "IJBPLPKCGHCKCNENHIBPNEIINHELGDJHEBEKPIGGGOBHGDNHEGIBOGGFBMOECGMABONDOADCGMHGJNKOENFBDKAEHGJOCIILFPLOKODMEEFFBEHBCFJMJNLCMACJFBNDELENGOKFPAMFHKDENLEAJDKICEIPHNKLMHMMOIIPBKJNOKBOLKAGAALGHKMJNFBEHBFHOINDLHFBCHLAAFBOFKIGJAKCAGONDAHNAHBBEIEOLGGKOOIEEDONEPBFJNMF",
    "status": "Ok"
}
```

The avatarCode will be obtained from readily available API.