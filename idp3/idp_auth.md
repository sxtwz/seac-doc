# 2.3.2 认证配置和属性对接-LDAP

本文中的 `idp.home` 均指代实际的 idp 安装路径。例如 `/opt/shibboleth-idp`

### LDAP 认证配置
修改 `/idp.home/conf/ldap.properties`
```bash
# ldap 的认证模式，通常使用 bindSearchAuthenticator。即通过管理员账号 bindDN 和 bindDNCredential，根据 userFilter 先搜所出用户的 dn，再和用户的密码进行验证
idp.authn.LDAP.authenticator = bindSearchAuthenticator
idp.authn.LDAP.ldapURL = ldap://ldap服务器IP地址
idp.authn.LDAP.useStartTLS  = false
idp.authn.LDAP.sslConfig = jvmTrust
# 开始搜索的 basedn
idp.authn.LDAP.baseDN = ou=People,dc=Test,dc=Test
# 有搜索权限的管理员账号
idp.authn.LDAP.bindDN = cn=Manager,dc=Test,dc=Test
# 管理员账号的密码
idp.authn.LDAP.bindDNCredential = 密码
# 允许递归搜索子树
idp.authn.LDAP.subtreeSearch = true
# 即用户名所对应的 ldap 属性。例如在 ldap 中通常是 uid，在 ad 中则需要修改为 (sAMAccountName={user})
idp.authn.LDAP.userFilter = (uid={user})
# 类似上一条，在 ad 中需要修改为 (sAMAccountName=$resolutionContext.principal)
idp.attribute.resolver.LDAP.searchFilter = (uid=$resolutionContext.principal)
```

#### LDAP 测试
由于 ldap 配置非常关键，建议先通过工具，在 IdP 环境上测试以下 LDAP 的认证，属性等是否均正常。以简化后续的排错判断。可以使用 [ldap-test-tool](https://github.com/shanghai-edu/ldap-test-tool) 进行测试

##### 安装 ldap-test-tool

[下载](https://github.com/shanghai-edu/ldap-test-tool/releases) ldap-test-tool 工具，选择对应的版本

解压后，修改 `cfg.json` 文件，填写 ldap 配置.`attributes` 指返回的属性，如果留空的话，则表示返回所有能读到的属性
```
{
    "ldap": {
        "addr": "ldap.example.org:389",
        "baseDn": "dc=example,dc=org",
        "bindDn": "cn=manager,dc=example,dc=org",
        "bindPass": "password",
        "authFilter": "(&(uid=%s))",
        "attributes": ["uid", "cn", "mail"],
        "tls":        false,
        "startTLS":   false
    },
    "http": {
        "listen": "0.0.0.0:8888"
    }
}
```

##### 查询测试
```
# ./ldap-test-tool search user qfeng
LDAP Search Start 
==================================


DN: uid=qfeng,ou=people,dc=example,dc=org
Attributes:
 -- uid  : qfeng 
 -- cn   : 冯骐测试 
 -- mail : qfeng@example.org


==================================
LDAP Search Finished, Time Usage 44.711268ms 
```

##### 认证测试
```
./ldap-test-tool auth single qfeng 123456
LDAP Auth Start 
==================================

qfeng auth test successed 

==================================
LDAP Auth Finished, Time Usage 47.821884ms 
```

更多信息请访问 [ldap-test-tool](https://github.com/shanghai-edu/ldap-test-tool) 查看

##### 属性对接
用以下内容替换`attribute-resolver.xml`文件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<AttributeResolver
        xmlns="urn:mace:shibboleth:2.0:resolver"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:schemaLocation="urn:mace:shibboleth:2.0:resolver http://shibboleth.net/schema/idp/shibboleth-attribute-resolver.xsd">

   <AttributeDefinition xsi:type="Simple" id="uid">
        <InputDataConnector ref="myLDAP" attributeNames="sAMAccountName"/>
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:uid" encodeType="false" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:2.5.4.2" friendlyName="uid" encodeType="false" />
   </AttributeDefinition>

   <AttributeDefinition xsi:type="Simple" id="cn">
        <InputDataConnector ref="myLDAP" attributeNames="displayName"/>
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:cn" encodeType="false" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:2.5.4.3" friendlyName="cn" encodeType="false" />
   </AttributeDefinition>

   <AttributeDefinition xsi:type="Simple" id="domainName">
        <InputDataConnector ref="staticAttributes" attributeNames="domainName"/>
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:domainName" encodeType="false" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:2.5.4.5" friendlyName="domainName" encodeType="false" />
   </AttributeDefinition>


   <AttributeDefinition xsi:type="ScriptedAttribute" id="typeOf">
        <InputDataConnector ref="myLDAP" attributeNames="userType"/>
        <Script><![CDATA[
        var localpart = "";
        if(userType.getValues().get(0)=="教师") localpart = "teacher";
        else if(userType.getValues().get(0)=="全日制学生") localpart = "student";
        else localpart = "other";
        typeOf.addValue(localpart);
            ]]></Script>
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:typeOf" encodeType="false" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:2.5.4.100.2" friendlyName="typeOf" encodeType="false" />
    </AttributeDefinition>

	<DataConnector id="staticAttributes" xsi:type="Static">
		<Attribute id="domainName">
			<Value>xxx.edu.cn</Value>
		</Attribute>
	</DataConnector> 

   	<DataConnector id="myLDAP" xsi:type="LDAPDirectory"
    	ldapURL="%{idp.attribute.resolver.LDAP.ldapURL}"
	    baseDN="%{idp.attribute.resolver.LDAP.baseDN}" 
    	principal="%{idp.attribute.resolver.LDAP.bindDN}"
	    principalCredential="%{idp.attribute.resolver.LDAP.bindDNCredential}"
    	connectTimeout="%{idp.attribute.resolver.LDAP.connectTimeout}"
	    responseTimeout="%{idp.attribute.resolver.LDAP.responseTimeout}">
    	<FilterTemplate>
        	<![CDATA[
            	%{idp.attribute.resolver.LDAP.searchFilter}
        	]]>
    	</FilterTemplate>
    	<ConnectionPool
        	minPoolSize="%{idp.pool.LDAP.minSize:3}"
        	maxPoolSize="%{idp.pool.LDAP.maxSize:10}"
        	blockWaitTime="%{idp.pool.LDAP.blockWaitTime:PT3S}"
        	validatePeriodically="%{idp.pool.LDAP.validatePeriodically:true}"
        	validateTimerPeriod="%{idp.pool.LDAP.validatePeriod:PT5M}"
        	expirationTime="%{idp.pool.LDAP.idleTime:PT10M}"
        	failFastInitialize="%{idp.pool.LDAP.failFastInitialize:false}" />
   	</DataConnector>
</AttributeResolver>
```