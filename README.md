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
