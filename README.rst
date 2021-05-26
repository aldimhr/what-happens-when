What happens when...
====================

Repositori ini merupakan jawaban untuk pertanyaan "Apa yang terjadi ketika 
mengetik google.com pada address box browser dan menekan enter?"

Berbeda dengan cerita pada umumnya, kami akan menocba menjawab pertanyaan ini
sedetail mungkin. Tidak boleh melewatkan apa pun.

Ini merupakan proses kolaboratif, jadi gali dan coba bantu! Ada banyak
sekali informasi detail yang belum ditulis, menunggu anda untuk
menambahkannya! Jadi tolong kirimkan kami pull request!

Ini semua berlisensi dibawah persyaratan lisensi `Creative Commons Zero`_ license.

Read this in `简体中文`_ (simplified Chinese), `日本語`_ (Japanese) and `한국어`_
(Korean). NOTE: these have not been reviewed by the alex/what-happens-when
maintainers.

Table of Contents
====================

.. contents::
   :backlinks: none
   :local:

Tombol "g" ditekan
----------------------
Pada bagian ini menjelaskan tidakan fisik pada keyboard dan OS INTERRUPTS. 
Saat anda menekan tombol "g", browser menerima EVENT dan fungsi auto-complete dimulai.
Bergantung pada algoritme browser anda dan apakah anda masuk mode 
pribadi/penyamaran atau tidak, berbagai saran akan disajikan kepada anda
di menu dropdown dibawah bilah URL. Sebagian besar algoritme ini
menyortir dan memprioritaskan hasil berdasarkan riwayat penelusuran, bookmark,
cookie, dan penelusuran populer dari internet secara keseluruhan.
Saat anda mengetik "google.com", banyak blok kode yang dijalankan dan saran
akan terus disempurnakan setiap penekanan tombol. Bahkan mungkin menyarankan
"google.com" sebelum anda selesai mengetiknya.

Tombol "enter" selesai ditekan
---------------------------

Untuk mengambil titik nol, mari pilih tombol Enter pada keyboard yang
menyentuh bagian paling bawah jangkauannya. Pada titik ini, rangkaian
listrik khusus untuk tombol enter ditutup (baik secara langsung, atau
secara kapasitif). Hal ini memungkinkan sejumlah kecil arus mengalir
ke sirkuit logika keyboard, yang memindai status setiap key switch
(tombol keyboard), menghilangkan gangguan listrik dari penutupan sakelar
yang terputus-putus, dan mengubahnya menjadi keycode integer, dalam 
hal ini 13. Pengontrol keyboard kemudian mengkodekan kode kunci untuk
dikirim ke komputer. Saat ini secara universal melalui koneksi Universal
Serial Bus (USB) atau Bluetooth, tetapi secara historis telah melalui
PS/2 atau ADB.

*Dalam kasus Keyboard USB:*

- Sirkuit USB keyboard ditenagai dengan suplai 5V yang disediakan melalui 1 pin 
  dari pengontrol host USB komputer.

- Keycode yang dihasilkan disimpan oleh memori sirkuit keyboard internal dalam
  register yang disebut "endpoint".

- Host pengontrol USB mengumpulkan "endpoint" setiap ~10ms (nilai minimum yang 
  telah dinyatakan (declare) oleh keyboard), sehingga ia akan mendapat keycode 
  yang disimpan di dalamnya.

- Nilai ini masuk ke USB SIE (Serial Interface Engine) untuk dikonversi dalam 
  satu atau lebih paket USB yang mengikuti protokol USB low-level.

- Paket-paket tersebut dikirim oleh sinyal listrik diferensial melalui pin D+
  dan D- (tengah 2) dengan kecepatan maksimum 1,5 Mb/s, karena perangkat HID
  (Human Interface Device) selalu dinyatakan sebagai "low-speed device"
  (USB 2.0 compliance).

- Sinyal serial ini kemudian diterjemahkan ke host pengontrol USB komputer, dan
  diinterpretasikan oleh driver perangkat keyboard universal Human Interface
  Device (HID) komputer. Value dari key tersebut kemudian diteruskan ke lapisan
  abstraksi perangkat keras sistem operasi.

*Dalam kasus Keyboard Virtual (seperti pada perangkat layar sentuh):*

- Saat pengguna meletakkan jarinya pada layar sentuh kapasitif modern,
  sejumlah kecil arus akan ditransfer ke jari. Ini menyelesaikan rangkaian
  melalui medan elektrostatis lapisan konduktif dan menciptakan penurunan
  tegangan pada titik tersebut di layar. ``Screen controller`` kemudian
  memunculkan interrupt yang melaporkan kordinat penekanan tombol.

- Kemudian OS seluler memberi tahu aplikasi yang sedang difokuskan dari 
  menekan event di salah satu elemen GUI-nya (yang sekarang adalah tombol
  aplikasi keyboard virtual)

- Keyboard virtual sekarang dapat raise interrupt perangkat lunak
  untuk mengirim pesan 'tombol ditekan' kembali ke OS.

- interrupt ini memberi tahu aplikasi yang sedang difokuskan dari event 'tombol ditekan'

Interrupt fires [BUKAN untuk keyboard USB]
---------------------------------------

Keyboard mengirimkan sinyal pada interrupt request line (IRQ), yang dipetakan
ke ``interrupt vector`` (integer) oleh pengontrol interrupt. CPU menggunakan
``Interrupt Descriptor Table`` (IDT) untuk memetakan vektor interrupt ke fungsi 
(``interrupt handlers``) yang disediakan oleh kernel. Ketika interrupt tiba,
CPU mengindeks IDT dengan vektor interrupt dan menjalankan penanganan yang sesuai.
Jadi, kernel dimasukkan.

(Di Windows) Pesan ``VM_KEYDOWN`` dikirim ke aplikasi.
--------------------------------------------------------

HID transport melewati key down event ke ``KBDHID.sys`` driver yang mengubah
penggunaan HID menjadi scancode. Dalam hal ini, scancode adalah ``VK_RETURN`` (``0x0D``).
Driver interface ``KBDHID.sys`` berinteraksi dengan ``KBDCLASS.sys`` (keyboard class driver). 
Driver ini menangani semua input keyboard dan keypad dengan cara yang aman.
Kemudian hal tersebut akan memanggil ``Win32K.sys`` (setelah berpotensi menyampaikan
pesan melalui filter keyboard pihak ketiga yang diinstal). Ini semua terjadi
dalam mode kernel.

``Win32K.sys`` mencari tahu window apa yang merupakan window aktif melalui
API ``GetForegroundWindow()``. API ini menyediakan window handle dari kotak
alamat browser. Window utama "message pump" kemudian memanggil 
``SendMessage(hWnd, WM_KEYDOWN, VK_RETURN, lParam)``. ``lParam`` merupakan 
bitmask yang menunjukkan informasi lebih lanjut tentang penekanan tombol:
repeat count (0 dalam hal ini), scancode (dapat bergantung pada OEM, tetapi
umumnya tidak untuk ``VK_RETURN``), baik tombol yang extended (alt, shift, ctrl)
juga ditekan (), dan beberapa status lainnya.

API window ``SendMessage`` adalah fungsi langsung yang menambahkan pesan ke 
antrian untuk window handle (``hWnd``). Kemudian, fungsi pemrosesan pesan utama
(disebut ``WindowProc``) dipanggil untuk memproses setiap pesan dalam antrian.

Window (``hWnd``) yang aktif sebenarnya adalah kontrol edit dan ``WindowProc``
dalam hal ini memiliki penanganan pesan untuk pesan ``WM_KEYDOWN``. Kode ini 
mencari di dalam parameter ke-3 yang diteruskan ke ``SendMessage`` (``wParam``)
dan, karena ``VK_RETURN`` tahu pengguna telah menekan tombol ENTER.

(Di OS X) NSEvent ``KeyDown`` dikirim ke aplikasi
--------------------------------------------------

Sinyal interrupt memicu kejadian interrupt event pada driver keyboard I/O Kit Kex.
Driver menerjemahkan sinyal menjadi keycode yang diteruskan ke proses OS X ``WindowServer``.
Akibatnya, ``WindowServer`` mengirimkan acara ke aplikasi yang sesuai (misalnya aktif atau listening)
melalui port Mach mereka di mana ia ditempatkan ke dalam antrian event. Event kemudian dapat
dibaca dari antrian ini oleh threads dengan sufficient privileges memanggil fungsi ``mach_ipc_dispatch``.
Hal ini paling sering terjadi occurs, dan ditangai oleh loop event utama ``NSApplication``, melalui 
``NSEvent`` dari ``NSEventType`` ``KeyDown``.

(Pada GNU / Linux) server Xorg mendengarkan keycodes
---------------------------------------------------

Ketika grafik ``X server`` digunakan, ``X`` akan menggunakan driver event generik
``evdev`` untuk mendapatkan penekanan tombol. Pemetaan ulang keycodes ke scancodes
dibuat dengan peta kunci dan aturan khusus ``X server``. Ketika pemetaan scancode
dari tombol yang ditekan selesai, ``X server`` mengirim karakter ke ``window manager``
paga gilirannya mengirim karakter ke window yang difokuskan. API grafis dari window
yang menerima karakter mencetak simbol font yang sesuai di bidang fokus yang sesuai.

Parse URL
---------

* Browser sekarang memiliki informasi berikut ini di URL (Uniform
  Resource Locator):

    - ``Protocol``  "http"
        Menggunakan 'Hyper Text Transfer Protocol'

    - ``Resource``  "/"
        Retrieve main (index) page


Apakah itu URL atau search term?
-----------------------------

Ketika tidak ada protocol atau nama domain yang valid diberikan,
browser melanjutkan untuk memasukkan teks yang dierikan dalam kotak
ke mesin pencari web default browser. Dalam banyak kasus, URL memiliki
bagian teks khusus yang ditambahkan padanya untuk memberi tahu mesin
pencari bahwa itu berasal dari bilah URL browser tertentu.

Ubah karakter Unicode non-ASCII di hostname
------------------------------------------------

* Browser memeriksa hostname untuk karakter yang tidak ada di ``a-z``,
  ``A-Z``, ``0-9``, ``-``, atau ``.``.
* Since the hostname is ``google.com`` there won't be any, but if there were
  the browser would apply `Punycode`_ encoding to the hostname portion of the
  URL.

Periksa daftar HSTS
---------------

* Browser memeriksa daftar "preloaded HSTS (HTTP Strict Transport
  Security)". Ini merupakan daftar situs web yang hanya dihubungi melalui
  HTTPS saja.
* Jika situs web ada dalam daftar, browser mengirimkan permintaanya melalui HTTPS,
  bukan HTTP. Jika tidak, permintaan awal dikirim melalui HTTP. 
  (Perhatikan bahwa situs web tetap dapat menggunakan kebijakan HSTS *tanpa* berada 
  dalam daftar HSTS. Permintaan HTTP pertama ke situs web oleh pengguna akan menerima 
  respons yang meminta agar pengguna hanya mengirim permintaan HTTPS. Namun, permintaan 
  HTTP tunggal ini berpotensi membuat pengguna rentan terhadap `downgrade attack`_,
  itulah sebabnya daftar HSTS disertakan di browser web modern.)

DNS lookup
----------

* Browser memeriksa apakah domain ada di cache-nya. (untuk melihat cache DNS di
  Chrome, buka `chrome://net-internals/#dns <chrome://net-internals/#dns>`_).
* Jika tidak ditemukan, browser akan memanggil fungsi pustaka ``gethostbyname``
  (berbeda-beda tergantung OS) untuk melakukan pencarian.
* ``gethostbyname`` memeriksa apakah hostname dapat diselesaikan dengan referensi
  di file ``hosts`` (lokasinya ``bervariasi tergantung OS yang digunakan``) sebelum
  mencoba menyelesaikan nama host melalui DNS.
* Jika ``gethostbyname`` tidak memiliki cache atau dapat menemukan di file ``hosts``
  maka dapat membuat permintaan ke server DNS yang dikonfigurasi di network stack.
  Biasanya ini adalah router lokal atau server DNS cache ISP.
* Jika server DNS berada di subnet yang sama, perpustakaan jaringan mengikuti di bawah
  ``ARP process`` untuk server DNS.
* Jika server DNS berada di subnet yang berbeda, network library mengikuti dibawah
  ``ARP process`` untuk default dateway IP.

Proses ARP
-----------

Untuk mengirim ARP (Address Resolution Protocol) broadcast, network stack library
memerlukan alamat IP target untuk dicari. Ini juga perlu mengetahua alamat MAC dari
antarmuka yang akan digunakan untuk mengirimkan ARP broadcast.

Cache ARP pertama kali diperiksa untuk entri ARP untuk IP target. Jika berada
di cache, fungsi library mengebalikan hasil: Target IP = MAC.

Jika entri tidak ada di cache ARP:

* Route table dicari, untuk melihat apakah alamat IP Target ada di salah satu subnet
  di route table lokal. Jika iya, pustaka menggunakan antarmuka yang terkait dengan
  subnet itu. Jika tidak, perpustakaan menggunakan antarmuka yang memiliki subnet 
  gateway default.

* Alamat MAC di antarmuka jaringan yang dipilih akan dicari.

* Network library mengirimkan permintaan ARP Layer 2 (data link
  layer dari `OSI model`_):

``ARP Request``::

    Sender MAC: interface:mac:address:here
    Sender IP: interface.ip.goes.here
    Target MAC: FF:FF:FF:FF:FF:FF (Broadcast)
    Target IP: target.ip.goes.here

Bergantung pada jenis perangkat keras antara komputer dan router:

Terhubung langsung:

* Jika komputer terhubung langsung ke router, respons router dengan ``ARP Reply``
  (lihat dibawah)

Hub:

* Jika komputer terhubung ke hub, hub akan mengirimkan permintaan ARP dari semua
  port lainnya. Jika router terhubung pada "kabel" yang sama, router akan merespons
  dengan ``ARP Reply`` (lihat di bawah).

Switch:

* Jika komputer terhubung ke sakelar, sakelar akan memeriksa CAM/MAC table lokalnya
  untuk melihat port mana yang memiliki alamat MAC yang kita cari. Jika sakelar
  tidak memiliki entri untuk alamat MAC, maka akan rebroadcast ARP request ke
  semua port lainnya.

* Jika sakelar memiliki entri di MAC/CAM table, ia akan mengirim permintaan ARP
  ke port yang memiliki alamat MAC yang kita cari.

* Jika router menggunakan "kabel" yang sama, router akan merespons dengan ``ARP Reply``
  (lihat di bawah).

``ARP Reply``::

    Sender MAC: target:mac:address:here
    Sender IP: target.ip.goes.here
    Target MAC: interface:mac:address:here
    Target IP: interface.ip.goes.here

Sekarang pustaka jaringan memiliki alamat IP baik dari server DNS atau gateway
default anda, ia dapat melanjutkan proses DNS-nya:

* DNS client membuat soket ke port UDP 53 di server DNS, menggunakan port sumber
  diatas 1023.
* Jika ukuran respons terlalu besar, TCP akan digunakan.
* Jika server DNS local / ISP tidak memiliki, maka pencarian rekursif diminta dan
  itu mengalir ke atas daftar server DNS sampai SOA tercapai, dan jika ditemukan
  jawaban maka akan dikembalikan.

Opening of a socket
-------------------
Setelah browser menerima alamat IP dari server tujuan, ia akan mengambilnya dan
nomor port yang diberikan dari URL (protokol HTTP default ke port 80, dan HTTPS
ke port 433), dan membuat panggilan ke fungsi perpustakaan sistem bernama ``socket``
dan meminta aliran soket TCP - ``AF_INET/AF_INET6`` dan ``SOCK_STREAM``.

* Permintaan ini pertama kali diteruskan ke Transport Layer tempat segmen TCP dibuat.
  Port tujuan ditambahkan ke header, dan port sumber dipilih dari dalam kisaran port
  dinamis kernel (ip_local_port_range in Linux).
* Segmen ini dikirim ke Network Layer, yang membungkus header IP tambahan. Alamat IP
  dari server tujuan serta mesin saay ini dimasukkan untuk membentuk sebuah paket.
* Paket selanjutnya tiba di Link Layer. Sebuah header bingkai ditambahkan yang menyertakan
  alamat MAC dari gateway (router lokal). Seperti sebelumnya, jika kernel tidak mengetahui
  alamat MAC dari gateway, kernal harus broadcast kuery ARP untuk menemukannya.

Pada titik ini paket siap untuk dikirim melalui:

* `Ethernet`_
* `WiFi`_
* `Cellular data network`_

Untuk sebagian besar koneksi internet rumahan atau bisnis kecil, paket akan melewati
komputer anda, mungkin melalui jaringan lokal, dan kemudian memalui modem (MOdulator / 
DEMolator) yang mengubah 1 dan 0 digital menjadi sinyal analog yang cocok untuk transmisi
melalui telepon, kabel, atau koneksi telepon nirkabel. Di ujung lain koneksi adalam modem
lain yang mengubah sinyal analog kembali menjadi data digital untuk di proses oleh `network
node`_ berikutnya dimana alamat dari dan ke akan dianalisis lebih lanjut.

Sebagian besar bisnis yang lebih besar dan beberapa koneksi perumahan yang lebih baru
akan memiliki koneksi fiber atau Ethernet langsung, dalam hal ini datanya tetap digital
dan diteruskan langsung ke `network node`_ berikutnya untuk di proses.

Akhirnya, paket tersebut akan mencapai router yang mengelola subnet lokal.
Dari sana, ia akan terus melakukan perjalanan ke router autonomous system's (AS) border, 
ASes lainnya, dan terakhir ke server tujuan. Setiap router di sepanjang
jalan mengekstrak alamat tujuan dari header IP dan mengarahkannya ke hop berikutnya
yang sesuai. Kolom Time to Live (TTL) di header IP dikurangi satu untuk setiap router
lewat. Paket akan dihapus jika bidang TTL mencapai nol atau jika router saat ini
tidak memiliki ruang dalam antriannya (mungkin karena kemacetan jaringan).

Pengiriman dan penerimaan ini terjadi beberapa kali setelah aliran koneksi TCP:

* Klien memilih initial sequence number (ISN) dan mengirim paket ke server
  dengan SYN bit set untuk menunjukkan pengaturan dari ISN.
* Server menerima SYN dan jika sedang dalam mood yang menyenangkan:
   * Server memilih nomor urut awalnya sendiri
   * Server menyetel SYN untuk menunjukkan ia memilih ISN-nya
   * Server menyalin (client ISN +1) ke ACK field dan menambahkan ACK flag
     untuk menunjukkan bahwa menerima paket pertama
* Klien acknowledges koneksi dengan mengirimkan paket:
   * Meningkatkan nomor urutnya sendiri
   * Meningkatkan receiver acknowledgment number
   * Setel ACK field
* Data ditransfer sebagai berikut:
   * Saat satu sisi mengirimkan N byte data, ia akan menginkatkan SEQ-nya dengan
     angka itu.
   * Ketika pihal lain acknowledges penerimaan paket itu (atau serangkaian paket),
     ia akan mengirimkan paket ACK dengan nilai ACK sama dengan urutan yang 
     diterima terkahir dari yang lain.
* Untuk menutup koneksi:
   * Semakin dekat mengirimkan paket FIN
   * Sisi lain ACK paket FIN dan mengirimkan FIN-nya sendiri
   * Semakin dekat acknowledges FIM pihak lain dengan ACK

TLS handshake
-------------
* The client computer sends a ``ClientHello`` message to the server with its
  Transport Layer Security (TLS) version, list of cipher algorithms and
  compression methods available.

* The server replies with a ``ServerHello`` message to the client with the
  TLS version, selected cipher, selected compression methods and the server's
  public certificate signed by a CA (Certificate Authority). The certificate
  contains a public key that will be used by the client to encrypt the rest of
  the handshake until a symmetric key can be agreed upon.

* The client verifies the server digital certificate against its list of
  trusted CAs. If trust can be established based on the CA, the client
  generates a string of pseudo-random bytes and encrypts this with the server's
  public key. These random bytes can be used to determine the symmetric key.

* The server decrypts the random bytes using its private key and uses these
  bytes to generate its own copy of the symmetric master key.

* The client sends a ``Finished`` message to the server, encrypting a hash of
  the transmission up to this point with the symmetric key.

* The server generates its own hash, and then decrypts the client-sent hash
  to verify that it matches. If it does, it sends its own ``Finished`` message
  to the client, also encrypted with the symmetric key.

* From now on the TLS session transmits the application (HTTP) data encrypted
  with the agreed symmetric key.

If a packet is dropped
----------------------

Sometimes, due to network congestion or flaky hardware connections, TLS packets
will be dropped before they get to their final destination. The sender then has
to decide how to react. The algorithm for this is called `TCP congestion
control`_. This varies depending on the sender; the most common algorithms are
`cubic`_ on newer operating systems and `New Reno`_ on almost all others.

* Client chooses a `congestion window`_ based on the `maximum segment size`_
  (MSS) of the connection.
* For each packet acknowledged, the window doubles in size until it reaches the
  'slow-start threshold'. In some implementations, this threshold is adaptive.
* After reaching the slow-start threshold, the window increases additively for
  each packet acknowledged. If a packet is dropped, the window reduces
  exponentially until another packet is acknowledged.

HTTP protocol
-------------

If the web browser used was written by Google, instead of sending an HTTP
request to retrieve the page, it will send a request to try and negotiate with
the server an "upgrade" from HTTP to the SPDY protocol.

If the client is using the HTTP protocol and does not support SPDY, it sends a
request to the server of the form::

    GET / HTTP/1.1
    Host: google.com
    Connection: close
    [other headers]

where ``[other headers]`` refers to a series of colon-separated key-value pairs
formatted as per the HTTP specification and separated by single newlines.
(This assumes the web browser being used doesn't have any bugs violating the
HTTP spec. This also assumes that the web browser is using ``HTTP/1.1``,
otherwise it may not include the ``Host`` header in the request and the version
specified in the ``GET`` request will either be ``HTTP/1.0`` or ``HTTP/0.9``.)

HTTP/1.1 defines the "close" connection option for the sender to signal that
the connection will be closed after completion of the response. For example,

    Connection: close

HTTP/1.1 applications that do not support persistent connections MUST include
the "close" connection option in every message.

After sending the request and headers, the web browser sends a single blank
newline to the server indicating that the content of the request is done.

The server responds with a response code denoting the status of the request and
responds with a response of the form::

    200 OK
    [response headers]

Followed by a single newline, and then sends a payload of the HTML content of
``www.google.com``. The server may then either close the connection, or if
headers sent by the client requested it, keep the connection open to be reused
for further requests.

If the HTTP headers sent by the web browser included sufficient information for
the webserver to determine if the version of the file cached by the web
browser has been unmodified since the last retrieval (ie. if the web browser
included an ``ETag`` header), it may instead respond with a request of
the form::

    304 Not Modified
    [response headers]

and no payload, and the web browser instead retrieve the HTML from its cache.

After parsing the HTML, the web browser (and server) repeats this process
for every resource (image, CSS, favicon.ico, etc) referenced by the HTML page,
except instead of ``GET / HTTP/1.1`` the request will be
``GET /$(URL relative to www.google.com) HTTP/1.1``.

If the HTML referenced a resource on a different domain than
``www.google.com``, the web browser goes back to the steps involved in
resolving the other domain, and follows all steps up to this point for that
domain. The ``Host`` header in the request will be set to the appropriate
server name instead of ``google.com``.

HTTP Server Request Handle
--------------------------
The HTTPD (HTTP Daemon) server is the one handling the requests/responses on
the server-side. The most common HTTPD servers are Apache or nginx for Linux
and IIS for Windows.

* The HTTPD (HTTP Daemon) receives the request.
* The server breaks down the request to the following parameters:
   * HTTP Request Method (either ``GET``, ``HEAD``, ``POST``, ``PUT``,
     ``PATCH``, ``DELETE``, ``CONNECT``, ``OPTIONS``, or ``TRACE``). In the
     case of a URL entered directly into the address bar, this will be ``GET``.
   * Domain, in this case - google.com.
   * Requested path/page, in this case - / (as no specific path/page was
     requested, / is the default path).
* The server verifies that there is a Virtual Host configured on the server
  that corresponds with google.com.
* The server verifies that google.com can accept GET requests.
* The server verifies that the client is allowed to use this method
  (by IP, authentication, etc.).
* If the server has a rewrite module installed (like mod_rewrite for Apache or
  URL Rewrite for IIS), it tries to match the request against one of the
  configured rules. If a matching rule is found, the server uses that rule to
  rewrite the request.
* The server goes to pull the content that corresponds with the request,
  in our case it will fall back to the index file, as "/" is the main file
  (some cases can override this, but this is the most common method).
* The server parses the file according to the handler. If Google
  is running on PHP, the server uses PHP to interpret the index file, and
  streams the output to the client.

Behind the scenes of the Browser
----------------------------------

Once the server supplies the resources (HTML, CSS, JS, images, etc.)
to the browser it undergoes the below process:

* Parsing - HTML, CSS, JS
* Rendering - Construct DOM Tree → Render Tree → Layout of Render Tree →
  Painting the render tree

Browser
-------

The browser's functionality is to present the web resource you choose, by
requesting it from the server and displaying it in the browser window.
The resource is usually an HTML document, but may also be a PDF,
image, or some other type of content. The location of the resource is
specified by the user using a URI (Uniform Resource Identifier).

The way the browser interprets and displays HTML files is specified
in the HTML and CSS specifications. These specifications are maintained
by the W3C (World Wide Web Consortium) organization, which is the
standards organization for the web.

Browser user interfaces have a lot in common with each other. Among the
common user interface elements are:

* An address bar for inserting a URI
* Back and forward buttons
* Bookmarking options
* Refresh and stop buttons for refreshing or stopping the loading of
  current documents
* Home button that takes you to your home page

**Browser High-Level Structure**

The components of the browsers are:

* **User interface:** The user interface includes the address bar,
  back/forward button, bookmarking menu, etc. Every part of the browser
  display except the window where you see the requested page.
* **Browser engine:** The browser engine marshals actions between the UI
  and the rendering engine.
* **Rendering engine:** The rendering engine is responsible for displaying
  requested content. For example if the requested content is HTML, the
  rendering engine parses HTML and CSS, and displays the parsed content on
  the screen.
* **Networking:** The networking handles network calls such as HTTP requests,
  using different implementations for different platforms behind a
  platform-independent interface.
* **UI backend:** The UI backend is used for drawing basic widgets like combo
  boxes and windows. This backend exposes a generic interface that is not
  platform-specific.
  Underneath it uses operating system user interface methods.
* **JavaScript engine:** The JavaScript engine is used to parse and
  execute JavaScript code.
* **Data storage:** The data storage is a persistence layer. The browser may
  need to save all sorts of data locally, such as cookies. Browsers also
  support storage mechanisms such as localStorage, IndexedDB, WebSQL and
  FileSystem.

HTML parsing
------------

The rendering engine starts getting the contents of the requested
document from the networking layer. This will usually be done in 8kB chunks.

The primary job of the HTML parser is to parse the HTML markup into a parse tree.

The output tree (the "parse tree") is a tree of DOM element and attribute
nodes. DOM is short for Document Object Model. It is the object presentation
of the HTML document and the interface of HTML elements to the outside world
like JavaScript. The root of the tree is the "Document" object. Prior to
any manipulation via scripting, the DOM has an almost one-to-one relation to
the markup.

**The parsing algorithm**

HTML cannot be parsed using the regular top-down or bottom-up parsers.

The reasons are:

* The forgiving nature of the language.
* The fact that browsers have traditional error tolerance to support well
  known cases of invalid HTML.
* The parsing process is reentrant. For other languages, the source doesn't
  change during parsing, but in HTML, dynamic code (such as script elements
  containing `document.write()` calls) can add extra tokens, so the parsing
  process actually modifies the input.

Unable to use the regular parsing techniques, the browser utilizes a custom
parser for parsing HTML. The parsing algorithm is described in
detail by the HTML5 specification.

The algorithm consists of two stages: tokenization and tree construction.

**Actions when the parsing is finished**

The browser begins fetching external resources linked to the page (CSS, images,
JavaScript files, etc.).

At this stage the browser marks the document as interactive and starts
parsing scripts that are in "deferred" mode: those that should be
executed after the document is parsed. The document state is
set to "complete" and a "load" event is fired.

Note there is never an "Invalid Syntax" error on an HTML page. Browsers fix
any invalid content and go on.

CSS interpretation
------------------

* Parse CSS files, ``<style>`` tag contents, and ``style`` attribute
  values using `"CSS lexical and syntax grammar"`_
* Each CSS file is parsed into a ``StyleSheet object``, where each object
  contains CSS rules with selectors and objects corresponding CSS grammar.
* A CSS parser can be top-down or bottom-up when a specific parser generator
  is used.

Page Rendering
--------------

* Create a 'Frame Tree' or 'Render Tree' by traversing the DOM nodes, and
  calculating the CSS style values for each node.
* Calculate the preferred width of each node in the 'Frame Tree' bottom-up
  by summing the preferred width of the child nodes and the node's
  horizontal margins, borders, and padding.
* Calculate the actual width of each node top-down by allocating each node's
  available width to its children.
* Calculate the height of each node bottom-up by applying text wrapping and
  summing the child node heights and the node's margins, borders, and padding.
* Calculate the coordinates of each node using the information calculated
  above.
* More complicated steps are taken when elements are ``floated``,
  positioned ``absolutely`` or ``relatively``, or other complex features
  are used. See
  http://dev.w3.org/csswg/css2/ and http://www.w3.org/Style/CSS/current-work
  for more details.
* Create layers to describe which parts of the page can be animated as a group
  without being re-rasterized. Each frame/render object is assigned to a layer.
* Textures are allocated for each layer of the page.
* The frame/render objects for each layer are traversed and drawing commands
  are executed for their respective layer. This may be rasterized by the CPU
  or drawn on the GPU directly using D2D/SkiaGL.
* All of the above steps may reuse calculated values from the last time the
  webpage was rendered, so that incremental changes require less work.
* The page layers are sent to the compositing process where they are combined
  with layers for other visible content like the browser chrome, iframes
  and addon panels.
* Final layer positions are computed and the composite commands are issued
  via Direct3D/OpenGL. The GPU command buffer(s) are flushed to the GPU for
  asynchronous rendering and the frame is sent to the window server.

GPU Rendering
-------------

* During the rendering process the graphical computing layers can use general
  purpose ``CPU`` or the graphical processor ``GPU`` as well.

* When using ``GPU`` for graphical rendering computations the graphical
  software layers split the task into multiple pieces, so it can take advantage
  of ``GPU`` massive parallelism for float point calculations required for
  the rendering process.


Window Server
-------------

Post-rendering and user-induced execution
-----------------------------------------

After rendering has been completed, the browser executes JavaScript code as a result
of some timing mechanism (such as a Google Doodle animation) or user
interaction (typing a query into the search box and receiving suggestions).
Plugins such as Flash or Java may execute as well, although not at this time on
the Google homepage. Scripts can cause additional network requests to be
performed, as well as modify the page or its layout, causing another round of
page rendering and painting.

.. _`Creative Commons Zero`: https://creativecommons.org/publicdomain/zero/1.0/
.. _`"CSS lexical and syntax grammar"`: http://www.w3.org/TR/CSS2/grammar.html
.. _`Punycode`: https://en.wikipedia.org/wiki/Punycode
.. _`Ethernet`: http://en.wikipedia.org/wiki/IEEE_802.3
.. _`WiFi`: https://en.wikipedia.org/wiki/IEEE_802.11
.. _`Cellular data network`: https://en.wikipedia.org/wiki/Cellular_data_communication_protocol
.. _`analog-to-digital converter`: https://en.wikipedia.org/wiki/Analog-to-digital_converter
.. _`network node`: https://en.wikipedia.org/wiki/Computer_network#Network_nodes
.. _`TCP congestion control`: https://en.wikipedia.org/wiki/TCP_congestion_control
.. _`cubic`: https://en.wikipedia.org/wiki/CUBIC_TCP
.. _`New Reno`: https://en.wikipedia.org/wiki/TCP_congestion_control#TCP_New_Reno
.. _`congestion window`: https://en.wikipedia.org/wiki/TCP_congestion_control#Congestion_window
.. _`maximum segment size`: https://en.wikipedia.org/wiki/Maximum_segment_size
.. _`varies by OS` : https://en.wikipedia.org/wiki/Hosts_%28file%29#Location_in_the_file_system
.. _`简体中文`: https://github.com/skyline75489/what-happens-when-zh_CN
.. _`한국어`: https://github.com/SantonyChoi/what-happens-when-KR
.. _`日本語`: https://github.com/tettttsuo/what-happens-when-JA
.. _`downgrade attack`: http://en.wikipedia.org/wiki/SSL_stripping
.. _`OSI Model`: https://en.wikipedia.org/wiki/OSI_model
