<h1>WAMPSERVER AD AUTHENTICATION</h1>
<p><h3>環境資訊</h3></p>
<p>OS：Windows Server 2012 R2 x64</p>
<p>Wampserver：3.2.6</p>
<p>Server：Apache 2.4</p>
<p>PHP：PHP 8.1</p>


||<p><h3>安裝步驟</h3></p><p>1. 安裝MOD_AUTHNZ_SSPI MODULE</p><p><h4>2. 檢查安裝是否成功</h4></p><p><h4>3. 補充</h4></p>|
| :-: | :- |



1. **安裝mod_authnz_sspi module**

下載 mod_authnz_sspi module

<https://www.apachehaus.net/modules/mod_authnz_sspi/>

然後解壓縮，將以下檔案放到 apache\modules

![](Wampserver Setting AD Authentication/002.png)

修改httpd.conf 

![](Wampserver Setting AD Authentication/003.png)

加入以下

```
LoadModule authnz_sspi_module modules/mod_authnz_sspi.so
```

修改httpd-vhosts.conf 

![](Wampserver Setting AD Authentication/004.png)

加入以下    
```
AuthName "My Intranet"
AuthType SSPI
SSPIAuth On
SSPIAuthoritative On

<RequireAll>
   <RequireAny>
      Require valid-sspi-user
   </RequireAny>
   <RequireNone>
      Require user "ANONYMOUS LOGON"
   </RequireNone>
</RequireAll>
```

like… :

![](Wampserver Setting AD Authentication/005.png)

查看是否安裝OK

![](Wampserver Setting AD Authentication/006.png)

**2.檢查安裝是否成功**

用以下PHP程式碼測試是否取得使用者資料

```
$cred = explode('\\',$_SERVER['REMOTE_USER']);

if (count($cred) == 1) array_unshift($cred, "(no domain info - perhaps SSPIOmitDomain is On)");

list($domain, $user) = $cred;

echo "User <B>$user</B><BR/>";

echo "logged into the Windows NT domain <B>$domain</B>";?>
```

如果用無痕模式就會跳出需登入

![](Wampserver Setting AD Authentication/007.png)

**3.補充**

**如果只是要用AD帳號進行登入就不需以上步驟，用如下程式碼進行登入並取撈AD資料即可**

```
  <?php
    $domain = 'ldap://192.168.1.1'; //設定網域名稱
    $dn = "dc=company,dc=com";
    
    // 連接至網域控制站
    $ldapconn = ldap_connect($domain) or die("無法連接至 $domain");
    print_r($ldapconn);
    
    $user = 'name'; //設定欲認證的帳號名稱
    $password = '123'; //設定欲認證的帳號密碼
    
    // 使用 ldap bind 
    $ldaprdn = $user . '@' . $domain; 
    // ldap rdn or dn 
    $ldappass = $password; // 設定密碼
    
    // 連接至網域控制站
    $ldapconn = ldap_connect($domain) or die("無法連接至 $domain"); 
    
    //以下兩行務必加上，否則 Windows AD 無法在不指定 OU 下，作搜尋的動作
    ldap_set_option($ldapconn, LDAP_OPT_PROTOCOL_VERSION, 3);
    ldap_set_option($ldapconn, LDAP_OPT_REFERRALS, 0);
    
    if ($ldapconn) 
    { 
        // binding to ldap server
        //echo("連結$domain <br>".$ldaprdn.",".$ldappass."<br>");
        $ldapbind = ldap_bind($ldapconn, $user, $ldappass);     
        //$ldapbind = ldap_bind($ldapconn, $ldaprdn, $ldappass);     
        //$ldapbind = ldap_bind($ldapconn);     
    
        // verify binding     
        if ($ldapbind) 
        {         
            $filter = "(sAMAccountName=name*)";
            $result = @ldap_search($ldapconn, $dn, $filter);  
            var_dump($result);
            if($result==false) 
                echo "認證失敗<br>";        
            else 
            {            
                echo "認證成功...<br>";             
                //取出帳號的所有資訊             
                $entries = ldap_get_entries($ldapconn, $result);
                $data = ldap_get_entries( $ldapconn, $result );
                //print_r($data);
                echo $data ["count"] . " entries returned\n";
                
                for($i = 0; $i <= $data ["count"]; $i ++) {
                    for($j = 0; $j <= $data [$i] ["count"]; $j ++) {
                        echo "[$i:$j]=".$data [$i] [$j] . ": " . $data [$i] [$data [$i] [$j]] [0] . "\n<br>";
                    }
                }                
            }    
        } 
        else 
        {         
            echo "認證失敗...<br>";     
        } 
    }
    
    echo 'Current script owner: ' . get_current_user();
    
    //關閉LDAP連結
    ldap_close($ldapconn);
    
}
?>

```

### 參考資料：
* [How to: Implementing Single Sign On on Windows with Apache](https://community.spiceworks.com/how_to/91377-implementing-single-sign-on-on-windows-with-apache)
* [Will there be a mod_auth_sspi for 2.4?](https://www.apachelounge.com/viewtopic.php?t=4548)
