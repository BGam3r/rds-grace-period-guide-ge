# RDS-ის 120-დღიანი Grace Period-ის განულება — Windows Server 2016 / 2019

ქართული პრაქტიკული ინსტრუქცია სატესტო Remote Desktop Session Host სერვერზე Grace Period-ის შესამოწმებლად და Registry Editor-ით გასანულებლად.

> **გამოყენების ფარგლები:** განკუთვნილია ლაბორატორიული გარემოსთვის. Grace Period-ის განულება არ ცვლის RDS CAL-ებს და არ ააქტიურებს Windows Server-ს. Microsoft ამ მიდგომას არ ურჩევს; სამუშაო გარემოში საჭიროა ლიცენზირების გამართული კონფიგურაცია.

## რეპოზიტორიის აღწერა

GitHub-ის **About → Description** ველისთვის:

RDS-ის 120-დღიანი Grace Period-ის განულების ქართული გზამკვლევი Windows Server 2016/2019-ის სატესტო გარემოსთვის: Registry, PowerShell-ით შემოწმება და პრობლემების დიაგნოსტიკა.

**სასურველი სახელი:** `rds-grace-period-guide-ge`

**Topics:** `windows-server`, `rds`, `remote-desktop-services`, `sysadmin`, `homelab`, `georgian`, `documentation`

## რას ეხება ინსტრუქცია

RD Session Host-ისთვის გათვალისწინებულია 120-დღიანი საწყისი პერიოდი. მისი ამოწურვისას მომხმარებლის სესიები შეიძლება ლიცენზირების შეცდომით დაიბლოკოს. ეს მექანიზმი Windows Server-ის Evaluation ვადასთან არ უნდა აგვერიოს.

ინსტრუქციის ძირითადი სამიზნეა Windows Server 2016 და 2019, გრაფიკული ინტერფეისით. ნაბიჯები სრულდება **RD Session Host-ზე**, არა უბრალოდ RD Licensing სერვერზე. კონკრეტულ სისტემაზე შედეგი უნდა გადაამოწმოთ; ეს რეპოზიტორია შესრულებული ტესტის ანგარიშს არ წარმოადგენს.

## 1. მომზადება

- დაგჭირდებათ ადმინისტრატორის უფლებები და სერვერის კონსოლზე წვდომა, მაგალითად vSphere/Hyper-V-დან.
- შექმენით აღდგენადი სარეზერვო ასლი. სატესტო VM-ზე მოკლევადიანი snapshot/checkpoint ცვლილების უკან დასაბრუნებლადაც გამოგადგებათ.
- დაგეგმეთ გადატვირთვა: აქტიური სესიები შეწყდება და შეუნახავი სამუშაო შეიძლება დაიკარგოს.
- ცვლილებამდე ჩაინიშნეთ გასაღების თავდაპირველი Owner და უფლებები. `.reg` ექსპორტი ACL-ისა და მფლობელის სარეზერვო ასლი არ არის.

## 2. დარჩენილი დღეების შემოწმება

სერვერზე გახსენით **Windows PowerShell → Run as administrator**:

```powershell
$settings = Get-CimInstance -Namespace 'root/CIMV2/TerminalServices' -ClassName Win32_TerminalServiceSetting
$result = Invoke-CimMethod -InputObject $settings -MethodName GetGracePeriodDays
$result | Select-Object ReturnValue, DaysLeft
```

წარმატებული გამოძახებისას `ReturnValue` უნდა იყოს `0`. `DaysLeft` აჩვენებს დარჩენილ დღეებს; `DaysLeft = 0` ნიშნავს ამოწურულ პერიოდს. შეცდომის შემთხვევაში შედეგი არ ჩათვალოთ ვადის ამოწურვის მტკიცებულებად.

## 3. რეესტრის გასაღების პოვნა და ექსპორტი

გაუშვით `regedit` ადმინისტრატორის უფლებებით და გადადით:

```text
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\RCM\GracePeriod
```

მარცხნივ მონიშნეთ `GracePeriod`, აირჩიეთ **File → Export**, მიუთითეთ **Selected branch** და შეინახეთ `.reg` ფაილი. თუ ექსპორტზე წვდომის შეცდომაა, შეასრულეთ შემდეგი ნაბიჯი და ექსპორტი წაშლამდე გაიმეორეთ.

## 4. უფლებების შეცვლა

1. `GracePeriod`-ზე დააჭირეთ მარჯვენა ღილაკს → **Permissions → Advanced**.
2. ჩაინიშნეთ არსებული **Owner** და უფლებები, შემდეგ Owner-ის გვერდით აირჩიეთ **Change**.
3. აირჩიეთ სერვერის ადმინისტრატორის ანგარიში ან შესაბამისი **Administrators** ჯგუფი; გამოიყენეთ **Check Names**.
4. დაადასტურეთ ცვლილება და დაბრუნდით Permissions ფანჯარაში.
5. არჩეულ ანგარიშს/ჯგუფს მიანიჭეთ **Full Control** უშუალოდ `GracePeriod` გასაღებზე და დააჭირეთ **Apply**.

წევრი სერვერის შემთხვევაში გადაამოწმეთ, რომ სწორ ადგილობრივ ჯგუფს ირჩევთ. სხვა ენაზე დაყენებულ Windows-ში ჯგუფის სახელი შეიძლება განსხვავდებოდეს. უფლებები არ გაავრცელოთ მშობელ `RCM` ან `Terminal Server` განყოფილებებზე.

## 5. Timebomb მნიშვნელობის წაშლა

მარჯვენა პანელში მოძებნეთ **REG_BINARY** მნიშვნელობა, რომლის სახელი იწყება ასე:

```text
L$RTMTIMEBOMB
```

დააჭირეთ მარჯვენა ღილაკს → **Delete → Yes**.

**წაშალეთ მხოლოდ ეს მნიშვნელობა.** `GracePeriod` გასაღების მთლიანად წაშლა ამ ინსტრუქციის ნაწილი არ არის. არ წაშალოთ `(Default)` ან სხვა ჩანაწერები. თუ მოსალოდნელი მნიშვნელობა არ ჩანს, შეჩერდით და გადაამოწმეთ სერვერის როლი და ბილიკი.

დასრულების შემდეგ აღადგინეთ თქვენ მიერ შეცვლილი უფლებები და Owner წინასწარ დაფიქსირებულ მდგომარეობამდე. არ წაშალოთ თავდაპირველად არსებული სისტემური ნებართვები.

## 6. სერვერის გადატვირთვა

დახურეთ Registry Editor და შეთანხმებულ დროს გადატვირთეთ სერვერი **Start → Power → Restart** გზით. წინასწარ დარწმუნდით, რომ მომხმარებლებმა სამუშაო შეინახეს.

## 7. შედეგის შემოწმება

ხელახლა გაუშვით მე-2 ნაბიჯის PowerShell ბრძანებები. მოსალოდნელი შედეგია განახლებული Grace Period, დაახლოებით 120 დღე. შემდეგ შეამოწმეთ სატესტო RDP კავშირი და, თუ ინსტრუმენტი ხელმისაწვდომია, **Server Manager → Tools → Remote Desktop Services → RD Licensing Diagnoser**.

თუ დღეები არ განახლდა ან კავშირი ისევ ვერ მყარდება, განმეორებით წაშლაზე გადასვლის ნაცვლად დიაგნოსტიკის შედეგი შეისწავლეთ.

## პრობლემების დიაგნოსტიკა

| სიმპტომი | შესამოწმებელი საკითხი |
| --- | --- |
| `Access is denied` | Regedit გაშვებულია თუ არა ადმინისტრატორით; სწორ ანგარიშს ეკუთვნის თუ არა გასაღები და აქვს თუ არა Full Control |
| `GracePeriod` ან მოსალოდნელი მნიშვნელობა არ ჩანს | მუშაობთ თუ არა სწორ RD Session Host-ზე და სწორ Registry ბილიკზე |
| PowerShell აბრუნებს Namespace/Class შეცდომას | როლისა და WMI პროვაიდერის ხელმისაწვდომობა; ბრძანების გაშვება უშუალოდ სამიზნე სერვერზე |
| დღეები განახლდა, მაგრამ RDP კვლავ არ მუშაობს | ქსელი, RDS სერვისები, მომხმარებლის უფლებები და RD Licensing Diagnoser-ის შეტყობინებები |
| ცვლილების შემდეგ ახალი პრობლემა გაჩნდა | გამოიყენეთ წინასწარ დაგეგმილი აღდგენა; მხოლოდ `.reg` იმპორტი უფლებებს ვერ აღადგენს |

## შენიშვნა წყაროს PowerShell სკრიპტზე

PDF-ში მოცემული ავტომატიზაცია უცვლელად არ არის გადმოტანილი:

- `takeown` ფაილებისა და დირექტორიების მფლობელობისთვისაა და `HKLM:\...` Registry ბილიკზე არ მუშაობს.
- წყაროს სკრიპტი მთლიან `GracePeriod` გასაღებს შლის, ხოლო ზემოთ აღწერილი პროცედურა მხოლოდ შესაბამის მნიშვნელობას ეხება.
- `TermService` არის Remote Desktop Services; მისი გადატვირთვა წყაროში შეცდომითაა აღწერილი როგორც Licensing სერვისის გადატვირთვა.

## სამუშაო გარემოსთვის

გამართეთ RD Licensing სერვერი, შესაბამისი RDS CAL-ები, ლიცენზირების რეჟიმი და Session Host-ის კავშირი ლიცენზიის სერვერთან. Grace Period-ის განულება მუდმივ გამოსავლად არ განიხილოთ.

## წყაროები

ინსტრუქცია მომზადებულია Brandon Lee-ის სტატიის მოწოდებული PDF-ის მიხედვით, დამატებითი ტექნიკური გადამოწმებით. ტექსტი ქართული ადაპტაციაა და არა სიტყვასიტყვითი თარგმანი.

- [Virtualization Howto — Reset 120 day RDS Grace period on 2016 and 2019](https://www.virtualizationhowto.com/2020/10/reset-120-day-rds-grace-period-on-2016-and-2019/) — ძირითადი წყარო; PDF-ში განახლების თარიღია 22 მარტი, 2024.
- [Microsoft — RDS Licensing troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/troubleshoot-rds-licensing-guidance)
- [Microsoft — GetGracePeriodDays](https://learn.microsoft.com/en-us/windows/win32/termserv/getgraceperioddays-win32-terminalservicesetting)
- [Microsoft — takeown](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/takeown)
- [Dell — RDS Grace Period reset](https://www.dell.com/support/kbdoc/en-us/000193714/how-to-reset-the-windows-remote-desktop-services-licensing-grace-period)

ქართული მასალა: [ITO.GE](https://ito.ge)
