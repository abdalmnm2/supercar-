// lib/main.dart
import 'dart:async';
import 'dart:math';

import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:intl/intl.dart';

void main() {
  runApp(const SuperCarApp());
}

/// ---------- MODELS ----------
class User {
  final int id;
  final String phone;
  String name;
  int points;
  User({required this.id, required this.phone, this.name = "", this.points = 0});
}

class Car {
  int id;
  String title;
  String description;
  double price;
  int sellerId;
  String status; // available, sold, in_auction
  Car({required this.id, required this.title, required this.description, required this.price, required this.sellerId, this.status = "available"});
}

class Order {
  int id;
  String type; // tow, service, delivery
  int userId;
  String pickup;
  String? destination;
  double price;
  double? currentLat;
  double? currentLng;
  String status;
  Order({required this.id, required this.type, required this.userId, required this.pickup, this.destination, this.price = 0.0, this.currentLat, this.currentLng, this.status = "requested"});
}

class Auction {
  int id;
  int carId;
  DateTime start;
  DateTime end;
  double highestBid;
  int? highestBidder;
  Auction({required this.id, required this.carId, required this.start, required this.end, this.highestBid = 0.0, this.highestBidder});
}

/// ---------- APP STATE (Provider) ----------
class AppState extends ChangeNotifier {
  User? currentUser;
  final List<Car> cars = [];
  final List<Order> orders = [];
  final List<Auction> auctions = [];
  int _userAutoInc = 1;
  int _carAutoInc = 1;
  int _orderAutoInc = 1;
  int _auctionAutoInc = 1;

  AppState() {
    // seed some demo data
    _seed();
  }

  void _seed() {
    cars.addAll([
      Car(id: _carAutoInc++, title: "تويوتا كامري 2017", description: "حالة ممتازة، فحص متوفر", price: 45000, sellerId: 0),
      Car(id: _carAutoInc++, title: "هيونداي النترا 2018", description: "استعمال خفيف، لا حوادث", price: 38000, sellerId: 0),
    ]);
    // create a demo user (id=1)
    currentUser = User(id: _userAutoInc++, phone: "0500000001", name: "مستخدم تجريبي", points: 50);
    notifyListeners();
  }

  Future<User> createOrGetGuest(String phone, {String name = ""}) async {
    // In real: verify via OTP. Here, create a simple user object and persist minimal
    if (currentUser != null && currentUser!.phone == phone) return currentUser!;
    currentUser = User(id: _userAutoInc++, phone: phone, name: name);
    notifyListeners();
    return currentUser!;
  }

  List<Car> listCars([String q = ""]) {
    if (q.isEmpty) return List.unmodifiable(cars.reversed);
    return cars.where((c) => c.title.contains(q) || c.description.contains(q)).toList();
  }

  Car addCar(String title, String description, double price, int sellerId) {
    final car = Car(id: _carAutoInc++, title: title, description: description, price: price, sellerId: sellerId);
    cars.add(car);
    notifyListeners();
    return car;
  }

  Order createOrder(String type, int userId, String pickup, {String? destination, double price = 0.0}) {
    final ord = Order(id: _orderAutoInc++, type: type, userId: userId, pickup: pickup, destination: destination, price: price);
    orders.add(ord);
    // start GPS simulation
    _simulateGps(ord);
    notifyListeners();
    return ord;
  }

  Auction createAuction(int carId, DateTime start, DateTime end) {
    final auc = Auction(id: _auctionAutoInc++, carId: carId, start: start, end: end);
    auctions.add(auc);
    notifyListeners();
    // schedule auction end
    Timer(end.difference(DateTime.now()), () {
      // finalize auction
      // in real: handle escrow, transfer, notifications
      notifyListeners();
    });
    return auc;
  }

  bool placeBid(int auctionId, int bidderId, double amount) {
    final a = auctions.firstWhere((x) => x.id == auctionId, orElse: () => throw Exception("Auction not found"));
    if (amount <= a.highestBid) return false;
    a.highestBid = amount;
    a.highestBidder = bidderId;
    notifyListeners();
    return true;
  }

  void _simulateGps(Order ord) {
    // simulate moving coordinates for 60 seconds
    double lat = 24.7136 + Random().nextDouble()*0.01;
    double lng = 46.6753 + Random().nextDouble()*0.01;
    Timer.periodic(const Duration(seconds:2), (t) {
      if (t.tick > 40) {
        t.cancel();
        ord.status = "delivered";
        notifyListeners();
        return;
      }
      lat += (Random().nextDouble()-0.5)*0.001;
      lng += (Random().nextDouble()-0.5)*0.001;
      ord.currentLat = lat;
      ord.currentLng = lng;
      ord.status = "on_route";
      notifyListeners();
    });
  }

  // Simple AI diagnosis stub (rules-based)
  Map<String,String> diagnose(String symptoms) {
    final s = symptoms.toLowerCase();
    if (s.contains("دخان") || s.contains("سخان") || s.contains("تسريب")) {
      return {"diagnosis":"احتمال خلل في نظام التبريد/تسريب","advice":"افحص الراديتير والوصلات، وتحقق من مستوى الزيت"};
    }
    if (s.contains("لا يعمل") || s.contains("لا يبدأ") || s.contains("لا يدور")) {
      return {"diagnosis":"قد تكون البطارية أو منظومة الاشتعال","advice":"اختبر البطارية، وفحص المولد والريموت إن وُجد"};
    }
    return {"diagnosis":"لا يمكن تحديد المشكلة بدون فحص ميداني","advice":"نوصي بطلب فحص ميداني عبر التطبيق"};
  }

  // Points system
  void addPoints(int userId, int pts) {
    if (currentUser != null && currentUser!.id == userId) {
      currentUser!.points += pts;
      notifyListeners();
    }
  }
}

/// ---------- APP UI ----------
class SuperCarApp extends StatelessWidget {
  const SuperCarApp({super.key});
  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider(
      create: (_) => AppState(),
      child: MaterialApp(
        title: 'SuperCar',
        theme: ThemeData(primarySwatch: Colors.blue, useMaterial3: true),
        home: const RootScreen(),
      ),
    );
  }
}

class RootScreen extends StatefulWidget {
  const RootScreen({Key? key}) : super(key: key);
  @override
  State<RootScreen> createState() => _RootScreenState();
}

class _RootScreenState extends State<RootScreen> {
  int _index = 0;
  @override
  Widget build(BuildContext context) {
    final pages = [
      const HomePage(),
      const MarketplacePage(),
      const OrdersPage(),
      const AuctionsPage(),
      const ProfilePage(),
    ];
    return Scaffold(
      body: pages[_index],
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _index,
        onTap: (i) => setState(() => _index = i),
        items: const [
          BottomNavigationBarItem(icon: Icon(Icons.home), label: "الرئيسية"),
          BottomNavigationBarItem(icon: Icon(Icons.store), label: "السوق"),
          BottomNavigationBarItem(icon: Icon(Icons.local_shipping), label: "خدمات"),
          BottomNavigationBarItem(icon: Icon(Icons.gavel), label: "المزادات"),
          BottomNavigationBarItem(icon: Icon(Icons.person), label: "حسابي"),
        ],
      ),
    );
  }
}

/// Home / Dashboard
class HomePage extends StatelessWidget {
  const HomePage({super.key});
  @override
  Widget build(BuildContext context) {
    final st = Provider.of<AppState>(context);
    final user = st.currentUser;
    return SafeArea(
      child: SingleChildScrollView(
        child: Padding(
          padding: const EdgeInsets.all(16.0),
          child: Column(
            children: [
              Row(children: [
                const Icon(Icons.directions_car, size: 40),
                const SizedBox(width: 10),
                Expanded(child: Text("مرحباً ${user?.name ?? 'ضيف'}", style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold))),
                Chip(label: Text("نقاط: ${user?.points ?? 0}"))
              ]),
              const SizedBox(height: 12),
              Card(
                child: ListTile(
                  title: const Text("طلب شاحنة سحب"),
                  subtitle: const Text("نقل سيارات متعطلة + متابعة حيّة عبر GPS"),
                  trailing: ElevatedButton(
                    child: const Text("طلب"),
                    onPressed: () {
                      Navigator.of(context).push(MaterialPageRoute(builder: (_) => const RequestTowPage()));
                    },
                  ),
                ),
              ),
              const SizedBox(height: 8),
              Card(
                child: ListTile(
                  title: const Text("طلب ميكانيكي متنقل"),
                  subtitle: const Text("خدمة صيانة في الموقع"),
                  trailing: ElevatedButton(onPressed: () {
                    Navigator.of(context).push(MaterialPageRoute(builder: (_) => const RequestServicePage()));
                  }, child: const Text("طلب")),
                ),
              ),
              const SizedBox(height: 8),
              Card(child: ListTile(title: const Text("تشخيص ذكي للأعطال (تجريبي)"), subtitle: const Text("وصف المشكلة سيعطيك تشخيصًا مبدئيًا"), trailing: ElevatedButton(onPressed: () {
                Navigator.of(context).push(MaterialPageRoute(builder: (_) => const AIDiagnosePage()));
              }, child: const Text("تشخيص")))),
              const SizedBox(height: 12),
              const Text("عروض سريعة", style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
              const SizedBox(height: 8),
              SizedBox(
                height: 220,
                child: ListView(
                  scrollDirection: Axis.horizontal,
                  children: st.listCars().map((c) => GestureDetector(
                    onTap: () => Navigator.of(context).push(MaterialPageRoute(builder: (_) => CarDetailPage(car: c))),
                    child: Container(width: 200, margin: const EdgeInsets.all(8), child: Card(
                      child: Padding(
                        padding: const EdgeInsets.all(8.0),
                        child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                          Text(c.title, style: const TextStyle(fontWeight: FontWeight.bold)),
                          const SizedBox(height: 8),
                          Text(c.description, maxLines: 3, overflow: TextOverflow.ellipsis),
                          const Spacer(),
                          Row(mainAxisAlignment: MainAxisAlignment.spaceBetween, children: [
                            Text(NumberFormat.currency(locale: "ar_SA", symbol: "﷼").format(c.price)),
                            ElevatedButton(onPressed: (){
                              Navigator.of(context).push(MaterialPageRoute(builder: (_) => SellFlowPage(car: c)));
                            }, child: const Text("تفاصيل"))
                          ])
                        ]),
                      ),
                    )),
                  )).toList(),
                ),
              )
            ],
          ),
        ),
      ),
    );
  }
}

/// Marketplace (list + add)
class MarketplacePage extends StatefulWidget {
  const MarketplacePage({Key? key}) : super(key: key);
  @override
  State<MarketplacePage> createState() => _MarketplacePageState();
}
class _MarketplacePageState extends State<MarketplacePage> {
  String q = "";
  @override
  Widget build(BuildContext context) {
    final st = Provider.of<AppState>(context);
    final results = st.listCars(q);
    return SafeArea(child: Column(children: [
      Padding(padding: const EdgeInsets.all(8.0), child: Row(children: [
        Expanded(child: TextField(decoration: const InputDecoration(hintText: "ابحث عن سيارة أو قطعة"), onChanged: (v)=> setState(()=> q=v))),
        IconButton(onPressed: () => Navigator.of(context).push(MaterialPageRoute(builder: (_) => const AddCarPage())), icon: const Icon(Icons.add))
      ])),
      Expanded(child: ListView(children: results.map((c)=>ListTile(title: Text(c.title), subtitle: Text(NumberFormat.currency(locale: "ar_SA", symbol: "﷼").format(c.price)), onTap: ()=> Navigator.of(context).push(MaterialPageRoute(builder: (_)=>CarDetailPage(car:c))))).toList()))
    ]));
  }
}

/// Orders / Services
class OrdersPage extends StatelessWidget {
  const OrdersPage({super.key});
  @override
  Widget build(BuildContext context) {
    final st = Provider.of<AppState>(context);
    return SafeArea(child: Padding(padding: const EdgeInsets.all(12), child: Column(children: [
      Row(mainAxisAlignment: MainAxisAlignment.spaceBetween, children: [const Text("طلباتك", style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)), ElevatedButton(onPressed: () {
        Navigator.of(context).push(MaterialPageRoute(builder: (_) => const RequestTowPage()));
      }, child: const Text("طلب شاحنة"))]),
      const SizedBox(height: 8),
      Expanded(child: ListView(children: st.orders.reversed.map((o) => Card(child: ListTile(title: Text(o.type), subtitle: Text("من: ${o.pickup}\nالحالة: ${o.status}"), trailing: (o.currentLat!=null)?Text("${o.currentLat!.toStringAsFixed(4)},${o.currentLng!.toStringAsFixed(4)}"):null))).toList()))
    ]));
  }
}

/// Auctions
class AuctionsPage extends StatefulWidget {
  const AuctionsPage({Key? key}) : super(key: key);
  @override
  State<AuctionsPage> createState() => _AuctionsPageState();
}
class _AuctionsPageState extends State<AuctionsPage> {
  @override
  Widget build(BuildContext context) {
    final st = Provider.of<AppState>(context);
    return SafeArea(child: Padding(padding: const EdgeInsets.all(12), child: Column(children: [
      Row(mainAxisAlignment: MainAxisAlignment.spaceBetween, children: [const Text("المزادات", style: TextStyle(fontSize: 18,fontWeight: FontWeight.bold)), ElevatedButton(onPressed: (){
        // create demo auction
        if (st.cars.isNotEmpty) {
          final car = st.cars.first;
          final now = DateTime.now();
          st.createAuction(car.id, now.add(const Duration(seconds:5)), now.add(const Duration(minutes:1)));
          ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text("تم إنشاء مزاد تجريبي (ينتهي بعد دقيقة)")));
        }
      }, child: const Text("إنشاء مزاد"))]),
      const SizedBox(height: 8),
      Expanded(child: ListView(children: st.auctions.map((a){
        final car = st.cars.firstWhere((c)=>c.id==a.carId, orElse: ()=>Car(id:0,title:"-","description":"",price:0.0,sellerId:0));
        return Card(child: ListTile(title: Text("مزاد: ${car.title}"), subtitle: Text("أعلى مزايدة: ${a.highestBid}"), trailing: ElevatedButton(onPressed: (){
          showDialog(context: context, builder: (_){
            double bid= a.highestBid+100;
            return AlertDialog(title: const Text("قدم مزايدة"), content: TextField(keyboardType: TextInputType.number, onChanged: (v)=>bid = double.tryParse(v) ?? bid, decoration: InputDecoration(hintText: "${a.highestBid+100}")), actions: [
              TextButton(onPressed: ()=> Navigator.of(context).pop(), child: const Text("إلغاء")),
              ElevatedButton(onPressed: (){
                final success = st.placeBid(a.id, st.currentUser!.id, bid);
                Navigator.of(context).pop();
                ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(success? "تمت المزايدة":"المزايدة أقل من الأعلى")) );
              }, child: const Text("تأكيد"))
            ]);
          });
        }, child: const Text("دخل المزاد"))));
      }).toList()))
    ]));
  }
}

/// Profile / Account
class ProfilePage extends StatelessWidget {
  const ProfilePage({super.key});
  @override
  Widget build(BuildContext context) {
    final st = Provider.of<AppState>(context);
    return SafeArea(child: Padding(padding: const EdgeInsets.all(12), child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
      Text("الهاتف: ${st.currentUser?.phone ?? '-'}"),
      Text("الاسم: ${st.currentUser?.name ?? '-'}"),
      const SizedBox(height: 12),
      ElevatedButton(onPressed: () => _showAbout(context), child: const Text("حول التطبيق")),
      const SizedBox(height: 8),
      ElevatedButton(onPressed: () => _logout(context), child: const Text("تسجيل خروج"))
    ])));
  }

  void _showAbout(BuildContext c) => showDialog(context: c, builder: (_)=>AlertDialog(title: const Text("عن سوبر كار"), content: const Text("MVP لتطبيق سوبر كار — نسخة تجريبية."), actions: [TextButton(onPressed: ()=>Navigator.of(c).pop(), child: const Text("إغلاق"))]));
  void _logout(BuildContext c) {
    final st = Provider.of<AppState>(c, listen: false);
    st.currentUser = null;
    st.notifyListeners();
    Navigator.of(c).pushReplacement(MaterialPageRoute(builder: (_) => const LoginFlowPage()));
  }
}

/// ---------- PAGES: Add Car / Car Detail / SellFlow ----------
class AddCarPage extends StatefulWidget {
  const AddCarPage({super.key});
  @override
  State<AddCarPage> createState() => _AddCarPageState();
}
class _AddCarPageState extends State<AddCarPage> {
  final title = TextEditingController();
  final price = TextEditingController();
  final desc = TextEditingController();
  @override
  Widget build(BuildContext context) {
    final st = Provider.of<AppState>(context, listen:false);
    return Scaffold(
      appBar: AppBar(title: const Text("أضف إعلان سيارة")),
      body: Padding(padding: const EdgeInsets.all(12), child: Column(children: [
        TextField(controller: title, decoration: const InputDecoration(labelText: "عنوان")),
        TextField(controller: price, decoration: const InputDecoration(labelText: "السعر"), keyboardType: TextInputType.number),
        TextField(controller: desc, decoration: const InputDecoration(labelText: "وصف")),
        const SizedBox(height: 12),
        ElevatedButton(onPressed: (){
          final t = title.text.trim();
          final p = double.tryParse(price.text) ?? 0.0;
          st.addCar(t, desc.text.trim(), p, st.currentUser?.id ?? 0);
          ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text("تم نشر الإعلان (تجريبي)")));
          Navigator.of(context).pop();
        }, child: const Text("نشر"))
      ])),
    );
  }
}

class AddCarButton extends StatelessWidget {
  const AddCarButton({super.key});
  @override
  Widget build(BuildContext context) => FloatingActionButton(child: const Icon(Icons.add), onPressed: () => Navigator.of(context).push(MaterialPageRoute(builder: (_) => const AddCarPage())));
}

class CarDetailPage extends StatelessWidget {
  final Car car;
  const CarDetailPage({super.key, required this.car});
  @override
  Widget build(BuildContext context) {
    return Scaffold(appBar: AppBar(title: Text(car.title)), body: Padding(padding: const EdgeInsets.all(12), child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
      Text(car.description), const SizedBox(height: 8),
      Text("السعر: ${NumberFormat.currency(locale: "ar_SA", symbol: "﷼").format(car.price)}"),
      const SizedBox(height: 12),
      ElevatedButton(onPressed: () {
        // simulate buy -> in real implement escrow and payments
        final st = Provider.of<AppState>(context, listen:false);
        st.addPoints(st.currentUser?.id ?? 0, 10);
        ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text("تم تسجيل عملية الشراء (تجريبي) - نقاط +10")));
      }, child: const Text("شراء (تجريبي)"))
    ]));
  }
}

class SellFlowPage extends StatelessWidget {
  final Car car;
  const SellFlowPage({super.key, required this.car});
  @override
  Widget build(BuildContext context) {
    return Scaffold(appBar: AppBar(title: const Text("تفاصيل الإعلان")), body: Padding(padding: const EdgeInsets.all(12), child: Column(children: [
      Text(car.title, style: const TextStyle(fontWeight: FontWeight.bold)),
      const SizedBox(height: 8),
      Text(car.description),
      const SizedBox(height: 8),
      Text("السعر: ${NumberFormat.currency(locale: "ar_SA", symbol: "﷼").format(car.price)}"),
      const Spacer(),
      ElevatedButton(onPressed: ()=> Navigator.of(context).pop(), child: const Text("الرجوع"))
    ])));
  }
}

/// ---------- Add Car Page alias used earlier ----------
class AddCarPageAlias extends AddCarPage {
  const AddCarPageAlias({super.key});
}

/// ---------- Request Tow Page (GPS simulation) ----------
class RequestTowPage extends StatefulWidget {
  const RequestTowPage({super.key});
  @override
  State<RequestTowPage> createState() => _RequestTowPageState();
}
class _RequestTowPageState extends State<RequestTowPage> {
  final addr = TextEditingController();
  @override
  Widget build(BuildContext context) {
    final st = Provider.of<AppState>(context);
    return Scaffold(
      appBar: AppBar(title: const Text("طلب شاحنة سحب")),
      body: Padding(padding: const EdgeInsets.all(12), child: Column(children: [
        TextField(controller: addr, decoration: const InputDecoration(labelText: "موقع التعطل (الوصف/العنوان)")),
        const SizedBox(height: 12),
        ElevatedButton(onPressed: (){
          final ord = st.createOrder("tow", st.currentUser?.id ?? 0, addr.text.trim(), price: 50.0);
          ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text("تم طلب الشاحنة. سيتم تحديث الموقع مباشرة.")));
          Navigator.of(context).push(MaterialPageRoute(builder: (_)=> OrderTrackingPage(orderId: ord.id)));
        }, child: const Text("طلب الآن"))
      ])),
    );
  }
}

class OrderTrackingPage extends StatelessWidget {
  final int orderId;
  const OrderTrackingPage({super.key, required this.orderId});
  @override
  Widget build(BuildContext context) {
    final st = Provider.of<AppState>(context);
    final ord = st.orders.firstWhere((o)=>o.id==orderId, orElse: ()=>Order(id:0,type:"",userId:0,pickup:"",price:0));
    return Scaffold(appBar: AppBar(title: const Text("متابعة الشاحنة")), body: Padding(padding: const EdgeInsets.all(12), child: Column(children: [
      Text("حالة الطلب: ${ord.status}"),
      const SizedBox(height: 8),
      if (ord.currentLat!=null) Text("موقع حالي: ${ord.currentLat!.toStringAsFixed(4)}, ${ord.currentLng!.toStringAsFixed(4)}"),
      const SizedBox(height: 12),
      Card(child: Padding(padding: const EdgeInsets.all(8), child: Column(children: [
        const Text("خريطة (محاكاة)"),
        const SizedBox(height: 8),
        Text("إحداثيات: ${ord.currentLat?.toStringAsFixed(4) ?? '-'} , ${ord.currentLng?.toStringAsFixed(4) ?? '-'}"),
      ])))
    ])));
  }
}

/// ---------- Request Service Page ----------
class RequestServicePage extends StatefulWidget {
  const RequestServicePage({super.key});
  @override
  State<RequestServicePage> createState() => _RequestServicePageState();
}
class _RequestServicePageState extends State<RequestServicePage> {
  final addr = TextEditingController();
  @override
  Widget build(BuildContext context) {
    final st = Provider.of<AppState>(context);
    return Scaffold(appBar: AppBar(title: const Text("طلب ميكانيكي متنقل")), body: Padding(padding: const EdgeInsets.all(12), child: Column(children: [
      TextField(controller: addr, decoration: const InputDecoration(labelText: "موقعك")),
      const SizedBox(height: 12),
      ElevatedButton(onPressed: (){
        final ord = st.createOrder("service", st.currentUser?.id ?? 0, addr.text.trim(), price: 120.0);
        ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text("تم طلب الخدمة، الميكانيكي في الطريق (تجريبي)")));
        Navigator.of(context).push(MaterialPageRoute(builder: (_)=> OrderTrackingPage(orderId: ord.id)));
      }, child: const Text("اطلب الآن"))
    ])));
  }
}

/// ---------- AI Diagnose Page ----------
class AIDiagnosePage extends StatefulWidget {
  const AIDiagnosePage({super.key});
  @override
  State<AIDiagnosePage> createState() => _AIDiagnosePageState();
}
class _AIDiagnosePageState extends State<AIDiagnosePage> {
  final symptoms = TextEditingController();
  Map<String,String>? result;
  @override
  Widget build(BuildContext context) {
    final st = Provider.of<AppState>(context);
    return Scaffold(appBar: AppBar(title: const Text("تشخيص الأعطال")), body: Padding(padding: const EdgeInsets.all(12), child: Column(children: [
      TextField(controller: symptoms, decoration: const InputDecoration(labelText: "اكتب وصف العطل...")),
      const SizedBox(height: 8),
      ElevatedButton(onPressed: (){
        final r = st.diagnose(symptoms.text);
        setState(()=> result = r);
      }, child: const Text("تحليل")),
      const SizedBox(height: 12),
      if (result!=null) Card(child: Padding(padding: const EdgeInsets.all(12), child: Column(children: [
        Text("تشخيص: ${result!['diagnosis']}", style: const TextStyle(fontWeight: FontWeight.bold)),
        const SizedBox(height: 6),
        Text("نصيحة: ${result!['advice']}")
      ])))
    ])));
  }
}

/// ---------- Login Flow Page ----------
class LoginFlowPage extends StatefulWidget {
  const LoginFlowPage({Key? key}) : super(key: key);
  @override
  State<LoginFlowPage> createState() => _LoginFlowPageState();
}
class _LoginFlowPageState extends State<LoginFlowPage> {
  final phone = TextEditingController();
  final name = TextEditingController();

  @override
  Widget build(BuildContext context) {
    final st = Provider.of<AppState>(context);
    return Scaffold(appBar: AppBar(title: const Text("تسجيل / دخول")), body: Padding(padding: const EdgeInsets.all(12), child: Column(children: [
      TextField(controller: phone, decoration: const InputDecoration(label: Text("رقم الجوال"))),
      TextField(controller: name, decoration: const InputDecoration(label: Text("الاسم (اختياري)"))),
      const SizedBox(height: 12),
      ElevatedButton(onPressed: () async {
        final u = await st.createOrGetGuest(phone.text.trim(), name: name.text.trim());
        Navigator.of(context).pushReplacement(MaterialPageRoute(builder: (_) => const RootScreen()));
      }, child: const Text("دخول"))
    ])));
  }
}
