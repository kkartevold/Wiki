## Oppgaver automatisering

 

### Fakturaavtale automatisk overføring - Gorgon, vaktmester

- Oppsett

- - Masterdata, leverandør 11702, automatisk overføring

  - Fakturaavtale, Vaktmester

  - - Tilknyttet leverandør 11702
    - Tilknyttet konteringsmal "Vaktmester" 
    - Fakturaavtalen overstyrer avdeling fra malen. Avdeling 2000 ligger i malen, mens fakturaavtalen bruker avdeling 2007.

- Utfør

- - I feilimport, endre fakturanummer på faktura som ligger på leverandør 11702 og utfør import.

- Resultat

- - Etter import vil faktura bli automatisk kontert og godkjent av "Invoice agreement user"
  - Faktura venter på overføring

  

### Automatisk godkjenning, konteringsmal og godkjenningsmal knyttet til leverandør ISS Facility Services

- Oppsett

- - Masterdata, leverandør 10621       "ISS Facility Services", automatisk godkjenning
  - Konteringsmal, "Kantine" tilknyttet leverandør 10621 "ISS Facility Services"
  - Godkjenningsmal, "Kantine", tilknyttet leverandør 10621 "ISS Facility Services" 

- Utfør

- - I feilimport, endre fakturanummer på faktura som ligger på leverandør 10621, fakturanummer       20181116A. Endre fakturanummer og utfør import.

- Resultat

- - Faktura ligger i status "Klar for overføring"
  - Faktura er godkjent av "System Auto Approval" og bruker som ble angitt som godkjenner       og sluttgodkjenner ligger som "Ferdig" i godkjenningsoppsettet på dokumentet og har dermed også søketilgang til fakturaen.

 

###  

### Konteringsmal tilknyttet leverandør og prosjekt. 

I dette scenariet er konteringsmalen har konteringsmalen flere kriterier og blir derfor valgt fremfor den som bare er knyttet til leverandør.

- Oppsett

- - Masterdata, leverandør 10621 "ISS Facility Services", automatisk godkjenning
  - Konteringsmal, "Kantine prosjekt 2020" tilknyttet leverandør 10621 "ISS Facility Services" og prosjekt "Eye-share 2020". Malen er fordelt på 3 avdelinger
  - Godkjenningsmal, "Kantine", tilknyttet leverandør 10621 "ISS Facility Services" 

- Utfør

- - I status "Godkjenning ikke startet", legg til konteringsmalen "Kantine" på en faktura tilhørende leverandør 10621, fakturanummer 20181114A
  - Angi en godkjenner og sluttgodkjenner på konteringslinjen.
  - Start godkjenningsflyt

- Resultat

- - Faktura ligger i status "Klar for overføring"
  - Faktura er godkjent av "System Auto Approval" og bruker som ble angitt som godkjenner og sluttgodkjenner ligger som "Ferdig" i godkjenningsoppsettet på dokumentet og har dermed også søketilgang til fakturaen.

 

 

### Hopp over status godkjent, etterkontroll av representasjon

- Oppsett

- - Admin, firma, innstillinger, faktura - godkjenning - Hopp over status godkjent
  - Aksjonsregel: egendefinert. Aksjon:  Stopp i status godkjent. Kriterier: Konto 7350

- Utfør

- - Konter en faktura på konto 7350 og godkjenn denne.

- Resultat

- - Faktura ligger i status godkjent og er merket med "Stop i status godkjent", "Representasjon"

 

### Aksjonsregel, mva kode endret

- Oppsett

- - Aksjonsregel: egendefinert. Aksjon:  Stopp i status godkjent.  Kriterier: Mvva (konteringsdimensjon) Operator: Er endret

- Utfør

- - Prekonter en faktura med en valgt mva kode. Send faktura til godkjenning til din bruker
  - Endre mva kode i konteringen og godkjenn faktura

- Resultat

- - Faktura ligger i status "Godkjent" og er merket med "Stopp i status godkjent", "Mva endret"