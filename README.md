## Patching istanbulkart

<p align="center">
  <img src="https://github.com/user-attachments/assets/8595ec28-7124-4ab2-a0b5-c73567dea963" />
</p>

istanbulkart app'inin telefonun sıfırlanması ve app'e tekrar giriş yapılması durumunda dijital ve fiziksel bütün kartlarınıza 'Cihazın eşleşmedi' hatası vererek 153'ü aramadan ulaşmanızı engelleyen bir mekanizması var.

App'i patchliyip bu kısıtlamayı kaldırmak istediğimde ilk varsaydığım şey device id kontrolünün muhtemelen server-sided olduğu idi ama şansıma durum böyle değilmiş. Bunu görebilmek için ilk denediğim şey [mitm-proxy](https://github.com/mitmproxy/mitmproxy) kullanarak API'ı incelemekti.


- Muhtemelen istanbulkart'ın son zamanlarda bir bankacılık uygulamasına kayıyor olmasından SSL pinning gibi basic korumalar implement edilmiş. Bir-iki yıl önce baktığımda onun bile olmadığını hatırlıyorum.

- istanbulkart'ın da kullanığı Android için HTTP client librarysi olan OkHttp'nin kendi içinde SSL pinning özelliği var ama pen testing yapmak isteyen çoğu kişinin bir noktada kırdığı bir şey olduğundan aşması çok kolay, kısa bir Google search ile keşfedilebiliyor.

API call'ları incelememden sonra cihaz eşleşmiyor olsa bile response içinde kart bilgisinin frontende ulaştığını görebildim.

Patchlemek için istanbulkart APK'sını indirdiğimizde bir base.apk ve diğer split APK'lardan bundle formatında olduğunu görebiliyoruz. İlgilendiğimiz kısım base.apk içinde olduğundan
[apktool](https://github.com/iBotPeaches/Apktool/releases/tag/v2.9.3) ile disassemble ediyoruz:  
```console
apktool d -r base.apk
```

MainActivity.smali'yi incelediğimizde bu instruction'ı görüyoruz:
```smali
invoke-super {p0, p1}, Lcom/facebook/react/ReactActivity;->onCreate(Landroid/os/Bundle;)V
```

ve istanbulkart'ın bir React Native app'i olduğunu anlıyoruz. React Native app'leri katlanılamayacak bir yavaşlıkta çalışmasının yanı sıra<sup>:)</sup> reverse engineerlaması da çok daha kolay. apktool'un apk'tan extract ettiği dosyalar içinde `assets/index.android.bundle`'a rastlıyoruz. Bu Facebook'un React Native'ı hızlandırmak için geliştirdiği (ve gayet başarısız) JavaScript bytecode'u olan Hermes bundle'ı... ve bunu disassemble etmemiz gerekiyor.

Bunun için [hbctool](https://github.com/bongtrop/hbctool) kullanıyoruz:
```console
hbctool disasm index.android.bundle hasm
```

hasm directory'sinde `instruction.hasm`'ı açarak patchlemek istediğimiz şey ile ilgili tamamen tahmini olarak keywordler aratıyoruz. Örneğin, hata mesajında gördüğümüz 'Cihazın eşleşmedi' textinden yola çıkarak 'match' aratıyoruz:
```rust
	LoadFromEnvironment 	Reg8:27, Reg8:6, UInt8:1
	GetById             	Reg8:27, Reg8:27, UInt8:23, UInt16:31628
	; Oper[3]: String(31628) 'digitalcard_unmatched_popup_title'
```

Byte code'da hata mesajını görüntülenmek üzere loadlayan instruction'ı buluyoruz. Aynı fonksiyonu incelemeye devam etmemiz üzerine muhtemelen API response'undan gelen hata kodlarına göre control flowu farklı adreslere dispatch eden koda rastlıyoruz:
```rust
	GetByIdShort        	Reg8:24, Reg8:24, UInt8:17, UInt8:46
	; Oper[3]: String(46) 'StatusCodes'

	GetById             	Reg8:24, Reg8:24, UInt8:22, UInt16:30666
	; Oper[3]: String(30666) 'ACCOUNT_PAIRED_DIFFERENT_DEVICE'

	JStrictNotEqualLong     Addr32:132, Reg8:25, Reg8:24
```
[Hermes byte code](https://p1sec.github.io/hermes-dec/opcodes_table.html)'u incelerseniz `JStrictNotEqualLong` instruction'ının `Reg8:25` ve `Reg8:24` registerlarındaki değerlerin strict olarak (JavaScript'teki `===` operatörü) eşit olmaması durumunda `Addr32:132` offsetine long jump yaptığını görebilirsiniz. Bizim aldığımız hatanın `ACCOUNT_PAIRED_DIFFERENT_DEVICE` olduğunu varsayarak bu instruction'ı aynı offsete sahip bir `unconditional long jump` ile değiştiriyorum:
```rust
	JmpLong Addr32:132
```

Bunu `ACCOUNT_PAIRED_DIFFERENT_DEVICE` veya `REGISTER_DEVICE_SUCCESS` gördüğüm her yerde uyguluyorum. Duruma göre bazı instructionları `REGISTER_DEVICE_SUCCESS`'ten uzağa jump yapmaması için comment-out yapmam da gerekebiliyor.
```rust
	; Oper[3]: String(30067) 'REGISTER_DEVICE_SUCCESS'
	; JStrictNotEqual     Addr8:71, Reg8:25, Reg8:24
```

Hepsini tamamladığımı umarak hbctool ile assembly etmeyi deniyorum:
```console
jhc@pop-os:~$ hbctool asm hasm index.android.bundle-patched
...
AssertionError: Overflowed instruction length is not supported yet.
```
hbctool bir fonksiyon içindeki instruction boyutları toplamının orijinal bundle'dakinden yüksek olmasını desteklemediğinden bu hatayı dönüyor. Neyse ki `132` offsetli unconditional long jumplar yaptığımızdan kırpabileceğimiz birçok instruction var. `JmpLong` girdiğim yerlerin altındaki kafama göre birkaç instruction'ı zaten hiçbir zaman ulaşılamayacağı için kaldırıyorum ve hbctool patchlediğim bundle'ı assemble ediyor:
```console
[*] Assemble 'hasm' to 'index.android.bundle-patched' path
[*] Hermes Bytecode [ Source Hash: 4e928e8c1c844e7f47ff8bcd603b20d0ae21e1e4, HBC Version: 76 ]
[*] Done
```

APK'yı yeniden buildlemeden önce runtime'da native liblerin yüklenememesi hatasını giderebilmek için AndroidManifest'i düzenliyorum:
```console
sed -i 's/android:extractNativeLibs="false"/android:extractNativeLibs="true"/' AndroidManifest.xml
```

ve bundle'ı apk'ya yerleştirerek apktool ile buildliyorum:
```console
cp -f index.android.bundle-patched base/assets/index.android.bundle
apktool b base -o base-patched.apk
```

Patchlenmiş base.apk ve diğer split APK'larımın hepsinin aynı imzaya sahip olması için hepsini aynı keystore ile signlamayı da unutmuyorum:
```console
apksigner sign --ks ks.keystore base-patched.apk
apksigner sign --ks ks.keystore split_config.*.apk
```

ve son olarak telefonuma adb ile kuruyorum:
```console
adb install-multiple base-patched.apk split_config.arm64_v8a.apk split_config.tr.apk \
						split_config.xxhdpi.apk
```

ve artık cihazın eşleşmedi hatası almıyorum :)

Bu "açığın" bir noktada kapatılacağını bilmekle beraber, bu zamana kadar nasıl server-sided bir check ile kaldığı da bir şaşkınlık kaynağı.

<p align="center">
  <img src="https://github.com/user-attachments/assets/05738a0c-0fc0-4d5d-b25c-81e77125cb7a" />
</p>
