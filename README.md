# 二部曲－最淺白的RCE攻擊，黑名單防禦大法
(參考資料：[splitline](https://github.com/splitline/py-sandbox-escape)課程講義)
在[首部曲－最淺白的RCE攻擊，用python sendbox 模擬實戰攻擊]()文章中有示範，毫無防護的sendbox，是可以輕易地下指令的，這時聰明的您一定會想到，那就把一些有安全疑慮的指令全部排除，不就能達到有效的防護了嗎？其實不盡然，就用簡單的範例帶領您了解。

## 黑名單防禦大法
```python
def check_secure(inp):
   　 block = [
     　'exec',
     　'open',
     　'file',
   　  'execfile',
 　    'import',
   　  'eval',
 　    'input',
  　   'hacker'
  　   # 以上為黑名單字串
  　   ]
  　 for s in block:
   　  if s in inp:
 　      raise Exception(“挨歐，'”+ s +”'是個非法字串喔！你想幹麻”)
 while True:
 　  try:
   　  inp = input('> ')
   　  check_secure(inp)
 　    ret = None
   　  exec(“ret=” + inp)
   　  if ret != None:
 　      print(ret)
 　  except Exception as e:
    　 print (e)
```
在check_securez方法中，檢查使用者輸入的內容中，有無包含exec、open等，高危險性的方法，這樣理論上就能有效的避免有心人士使用RCE，且這種有危險性的方法也不是很多，因此要手工建置這份黑名單，也不是一件困難的事情。執行的效果如下圖，exec這個方法就被拒之於門外，這樣是否達成百分之百的防護呢？
![非法字串執行結果](https://i.imgur.com/Wet90qa.png)
> 漏洞何在？
執行以下程式碼，竟然能夠成功import os這個方法（如下圖2）。使用`__builtins__`這個函式來檢視整個console中的環境，這個概念就如同「`import os`」也可以寫成「`__import__(‘os’)``」。
```python
__builtins__.__dict__['ex'+'ec']('imp'+'ort os')
```
![成功繞過黑名單](https://i.imgur.com/KprQ0cu.png)
其實「import」這個語法到python編譯時，還是會先轉換成「__import__」，因此其實兩者是相同的。而__builtins__又是什麼呢？這要從命名空間（Namespace）來說明，簡單來說就是命名變數，例如a=1就是命名a空間內容為1。

而命名空間（Namespace）有分為區域變數（local）與全域變數（glabol）兩種：
1. 區域變數（local）：只會在方法、檔案內可以使用這個變數。
2. 全域變數（glabol）：所有環境都可以使用。


![dir方法執行範例](https://i.imgur.com/AwxU4Fq.png)
不管任何程式語言都有許多保留字元，而保留字元一定都是全域變數（glabol），而這些保留字元在開啟一個新環境的時候，就已經準備好了，它準備在哪裡呢？就放在__builtins__模塊裡面，可以利用dir()來查看，若dir()內沒有參數，就會顯示目前環境的所有套件，若有顯示參數，則會顯示該參數的屬性（attribute），因此若輸入dir(__builtins__)，即可看到所有__builtins__底下的屬性，這些也是每個環境建制時，都會設定好的。
```py
dir(__builtins__)
```
![__builtins__的所有屬性](https://i.imgur.com/scsabwF.png)

找到exex方法的流程如下圖。這個原理是由於python物件都會互相繼承，而立用互相繼承的關聯性，取得想要使用的方法。首先在__builtins__當中找到dict變數的物件，這非常常見、合理，但該物件中有exex方法，因此由此可以藉此連結，取得exec方法的使用。
![找到exec方法的流程](https://i.imgur.com/QXBibUA.png)

記得在使用`__builtins__.__dict__`時，必須將後方的exec用字串相加的方式隔開，如範例中的’ex’+’ec’，這樣才能避免被黑名單發現，便能成功的繞過黑名單檢測。這也算是一個非常普遍的RCE。

> 那該如何防禦？
所謂道高尺，魔高一丈，由此可知，`__builtins__`成了黑名單防禦大法的一大漏洞，那如果在環境載入時，就把`__builtins__`當中的危險函數全部刪除，不就能補足這個漏洞了嗎？請見「三部曲－最淺白的RCE攻擊，誅九族大法」。

(參考資料：[splitline](https://github.com/splitline/py-sandbox-escape)課程講義)
