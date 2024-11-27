# Create user Active Directory
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
```