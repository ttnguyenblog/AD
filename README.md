## Kiểm tra hệ thống AD
```bash
# Lấy thông tin về tất cả các bộ điều khiển miền (Domain Controllers)
Get-ADDomainController -Filter *

# Đếm tổng số người dùng trong Active Directory
(Get-ADUser -Filter *).Count

# Đếm tổng số nhóm trong Active Directory
(Get-ADGroup -Filter *).Count

# Đếm tổng số máy tính trong Active Directory
(Get-ADComputer -Filter *).Count

# Hiển thị tóm tắt về trạng thái sao chép của AD (Active Directory)
repadmin /replsummary

# Kiểm tra trạng thái của các dịch vụ quan trọng trên Domain Controller
$Services='DNS','DFS Replication','Intersite Messaging','Kerberos Key Distribution Center','NetLogon','Active Directory Domain Services'
ForEach ($Service in $Services) {
    Get-Service $Service | Select-Object Name, Status
}

# Chạy kiểm tra DNS toàn diện trên các Domain Controllers
DCDiag /Test:DNS /e /v

# Đồng bộ tất cả các bản sao của AD trên Domain Controllers
repadmin /syncall /Aped

# Hiển thị thông tin về lần sao lưu cuối cùng của AD
repadmin /showbackup

# Hiển thị thông tin chi tiết về sao chép của tất cả các Domain Controllers
repadmin /showrepl *

# Lấy lại thông tin về tất cả các Domain Controllers (lệnh lặp lại)
Get-ADDomainController -Filter *
```

## Create user Active Directory
```bash
# Đường dẫn OU nơi người dùng sẽ được tạo
$ouPath = "cn=Users,dc=domain,dc=com,dc=vn"

# Mật khẩu mặc định cho các user
$password = "admin@123"

# Tạo 200 user
for ($i = 1; $i -le 200; $i++) {
    # Định dạng số thứ tự (user001, user002, ...)
    $userNumber = $i.ToString("D3")
    $username = "user$userNumber"
    $upn = "$username@domain.com.vn"
    $displayName = "User $userNumber"

    # Câu lệnh dsadd user
    $dsaddCommand = @"
dsadd user "cn=$username,$ouPath" -samid $username -upn $upn -display "$displayName" -disabled no -pwd "$password" -mustchpwd no
"@

    # In lệnh ra màn hình để kiểm tra
    Write-Output $dsaddCommand

    # Thực thi lệnh dsadd
    Invoke-Expression $dsaddCommand
}

# Get info user
get-aduser -filter * -Properties * | select displayname, name, UserPrincipalName | export-csv -path c:\export-all-2.csv
```

## Thu thập thông tin domain, name, IP, DNS các Domain Controller
```bash
(Get-ADForest).Domains | ForEach-Object {
    $domain = $_
    Get-ADDomainController -Filter * -Server $domain | ForEach-Object {
        $dnsServers = (Get-WmiObject -Class Win32_NetworkAdapterConfiguration -ComputerName $_.Name).DNSServerSearchOrder
        [PSCustomObject]@{
            Domain       = $domain
            Name         = $_.Name
            IPv4Address  = $_.IPv4Address
            DNSServer    = ($dnsServers -join " ")
        }
    }
} | Format-Table -AutoSize


Domain          Name    IPv4Address     DNSServer
domain1.local   DC1     192.168.1.10    8.8.8.8 8.8.4.4
domain2.local   DC2     192.168.2.20    8.8.8.8 1.1.1.1
```
## Chuyển đổi master role

```bash
netdom query fsmo

#chuyển fsmo về máy chủ AD-06
Move-ADDirectoryServerOperationMasterRole –Identity “AD-06” –OperationMasterRole 0,1,2,3,4
```

- **Schema master**: đây là zone quản lý Active Directory Schema, làm nhiệm vụ đồng bộ cấu trúc của Schema AD tới tất cả các domain. Đây là role duy nhất trên toàn forest.
- **Domain naming master**: đây cũng là zone duy nhất trên toàn forest có nhiệm vụ quản lý các tác vụ khi add thêm 1 child domain vào forest (VD quản lý không để add thêm 1 child domain có cùng “tên” với child domain khác)
- **RID master**: mỗi một domain đều sẽ có zone này, có tác dụng quản lý, cung cấp một dãy RIDs (relaytive IDs) là các mã định danh. Khi mất role RID bị mất hoặc chết thì hệ thống sẽ không thể tạo được thêm được user hoặc computer mới trong AD
- **PDC emulate master**: mỗi một domain đều sẽ có zone này. DC nắm PDC role sẽ được các member ưu tiên tìm đến để chứng thực đầu tiên. Zone PDC quản lý cả việc thay đổi password, thay đổi các Group Policy và quản lý đồng bộ thời gian trên từng domain
- **Infrastructure master**: có tác dụng đứng ra thực hiện các tác vụ “chuyển dịch” các GUID, SID và Distinguished Names giữa các domain. VD như khi thực hiện Add 1 user từ một domain vào một security group trong một domain khác thì Infratructe zone sẽ thực hiện. Hay phân quyền một user ở domain cha có quyền hạn trên tài nguyên của child domain thì role Infratrute này sẽ thực hiện.
