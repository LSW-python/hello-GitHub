# hello-GitHub
import hashlib

from django.http import JsonResponse, HttpResponse
from django.shortcuts import render, redirect

# Create your views here.
from django.urls import reverse

from App.models import MainWheel, MainNav, MainMustBuy, MainShop, MainShow, FoodType, Goods, User, Cart, Order


def home(request):
    wheels = MainWheel.objects.all()
    navs = MainNav.objects.all()
    mustbuys = MainMustBuy.objects.all()
    mainshops = MainShop.objects.all()
    mainshop0 = mainshops[0]
    mainshop1_2 = mainshops[1:3]
    mainshop3_6 = mainshops[3:7]
    mainshop7_10 = mainshops[7:11]
    mainshows = MainShow.objects.all()
    data = {
        "title": "首页",
        "wheels": wheels,
        "navs": navs,
        "mustbuys": mustbuys,
        "mainshop0": mainshop0,
        "mainshop1_2": mainshop1_2,
        "mainshop3_6": mainshop3_6,
        "mainshop7_10": mainshop7_10,
        "mainshows": mainshows,
    }
    return render(request, 'app/home/home.html', context=data)


def market(request):
    return redirect(reverse("app:marketWithParams", kwargs={"typeid": "104749", "childcid": '0', "sortrule": "0"}))


def marketWithParams(request, typeid, childcid, sortrule):
    foodtypes = FoodType.objects.all()
    goodsList = Goods.objects.filter(categoryid=typeid)

    if childcid != "0":
        goodsList = goodsList.filter(childcid=childcid)

    """
        sortrule 
            0 默认排序，综合排序
            1 销量降序
            2 销量升序
            3 价格降序
            4 价格升序
    """
    if sortrule == "1":
        goodsList = goodsList.order_by("productnum")
    elif sortrule == "2":
        goodsList = goodsList.order_by("-productnum")
    elif sortrule == "3":
        goodsList = goodsList.order_by("-price")
    elif sortrule == "4":
        goodsList = goodsList.order_by("price")

    foodtype = foodtypes.filter(typeid=typeid).first()

    # print(foodtype.childtypenames)
    childtypesTran = []
    # 全部分类: 0  # 进口水果:103534#国产水果:103533
    childtypes = foodtype.childtypenames.split("#")
    # ["全部分类:0","进口水果:103534","国产水果:103533"]
    for item in childtypes:
        itemchild = item.split(":")
        childtypesTran.append(itemchild)
    # [["全部分类","0"],["进口水果","103534“]...]

    data = {
        "title": "闪购",
        "foodtypes": foodtypes,
        "goodslist": goodsList,
        "childtypetran": childtypesTran,
        "typeid": typeid,
        "childcid": childcid,
    }
    return render(request, 'app/market/market.html', context=data)


def cart(request):
    username = request.session.get("username")
    if not username:
        return redirect(reverse("app:login"))

    user = User.objects.filter(u_name=username).first()
    # user = User()
    goodsList = user.cart_set.filter(c_belong=False)
    print(goodsList)

    data = {
        "title": "购物车",
        "goodsList": goodsList,
    }
    return render(request, 'app/cart/cart.html', context=data)


def mine(request):
    username = request.session.get("username")
    data = {
        "title": "我的",
    }
    if username:
        user = User.objects.get(u_name=username)
        print(user.u_icon.url)
        data["icon"] = user.u_icon.url
        data["username"] = username
        data["isLogin"] = "login"
        not_pay_num = Order.objects.filter(o_user=user).filter(o_status=0).count()
        not_receive_num = Order.objects.filter(o_user=user).filter(o_status=1).count()
        data["not_pay"] = not_pay_num
        data["not_receive"] = not_receive_num


    return render(request, 'app/mine/mine.html', context=data)


def root(request):
    return redirect(reverse("app:home"))


def register(request):
    if request.method == "POST":
        username = request.POST.get("username")
        password = request.POST.get("password")
        email = request.POST.get("email")
        icon = request.FILES["icon"]
        # 需要再次判断

        # 密码的安全
        password = password2Md5(password)
        user = User()
        user.u_name = username
        user.u_password = password
        user.u_email = email
        user.u_icon = icon
        user.save()
        response = redirect(reverse("app:mine"))
        request.session["username"] = username
        return response


    elif request.method == "GET":
        return render(request, 'app/mine/user/register.html')


def checkUser(request):
    username = request.GET.get("username")
    users = User.objects.filter(u_name=username)
    data = {
        "msg": "用户名可用",
        "status": "888",
    }
    # 900 状态码代表用户已存在
    if users.exists():
        data["msg"] = "用户名已存在"
        data["status"] = "900"
    return JsonResponse(data)


def password2Md5(password):
    md5 = hashlib.md5()
    md5.update(password.encode("utf-8"))
    return md5.hexdigest()


def logout(request):
    request.session.flush()
    return redirect(reverse("app:mine"))


def login(request):
    if request.method == "POST":
        username = request.POST.get("username")
        password = request.POST.get("password")
        users = User.objects.filter(isDelete=False).filter(u_name=username)
        if users.exists():
            user = users.first()
            u_password = user.u_password
            password = password2Md5(password)
            if u_password == password:
                request.session["username"] = username
                return redirect(reverse("app:mine"))

        return HttpResponse("用户名或密码错误")

    elif request.method == "GET":
        return render(request, 'app/mine/user/login.html')


def userinfo(request):
    if request.method == "POST":
        username = request.session.get("username")
        users = User.objects.filter(u_name=username)
        if users.exists():
            user = users.first()
            password = request.POST.get("password")
            if password:
                user.u_password = password2Md5(password)
            icon = request.FILES["icon"]
            if icon:
                user.u_icon = icon
            user.save()
            return redirect(reverse("app:mine"))
    elif request.method == "GET":
        return render(request, 'app/mine/user/userinfo.html')


def addToCart(request):
    username = request.session.get("username")
    data = {
        "status": "200",
        "msg": "ok",
    }
    if not username:
        data["status"] = "302"
        data["msg"] = "用户未登录"
        return JsonResponse(data)

    goodsid = request.GET.get("goodsid")
    goods = Goods.objects.filter(pk=goodsid).first()
    user = User.objects.filter(u_name=username).first()

    cart_item = Cart.objects.filter(c_user=user).filter(c_goods=goods).filter(c_belong=False).first()

    if not cart_item:
        cart_item = Cart()
    else:
        cart_item.c_num = cart_item.c_num + 1
    cart_item.c_goods = goods

    cart_item.c_user = user
    cart_item.save()
    data["c_num"] = cart_item.c_num

    return JsonResponse(data)


def subToCart(request):
    username = request.session.get("username")
    data = {
        "status": "200",
        "msg": "ok",
    }
    if not username:
        data["status"] = "302"
        data["msg"] = "用户未登录"
        return JsonResponse(data)

    user = User.objects.filter(u_name=username).first()

    goodsid = request.GET.get("goodsid")

    goods = Goods.objects.filter(pk=goodsid).first()

    carts = Cart.objects.filter(c_belong=False).filter(c_user=user).filter(c_goods=goods)

    if carts.exists():
        cart_item = carts.first()
        if cart_item.c_num == 1:
            cart_item.delete()
            data["num"] = 0
        else:
            cart_item.c_num = cart_item.c_num - 1
            cart_item.save()
            data["num"] = cart_item.c_num
    else:
        data["status"] = "202"
        data["msg"] = "操作数据不存在"
    return JsonResponse(data)


def changeCheck(request):
    cartid = request.GET.get("cartid")
    cart_item = Cart.objects.get(pk=cartid)
    cart_item.c_select = not cart_item.c_select
    cart_item.save()
    data = {
        "status": "200",
        "msg": "ok",
        "is_select": cart_item.c_select,
    }
    return JsonResponse(data)


def subCartGoods(request):
    cartid = request.GET.get("cartid")
    cart_item = Cart.objects.filter(pk=cartid).first()
    data = {
        "status": 200,
        "msg": "ok",
    }
    if cart_item.c_num == 1:
        cart_item.delete()
        data["num"] = 0
    else:
        cart_item.c_num = cart_item.c_num - 1
        cart_item.save()
        data["num"] = cart_item.c_num

    return JsonResponse(data)


def generateOrder(request):
    selects = request.GET.get("selects")
    select_list = selects.split("#")
    print(select_list)

    data = {
        "status": "200",
        "msg": "ok",
    }
    username = request.session.get("username")
    user = User.objects.get(u_name=username)
    order = Order()
    order.o_user = user
    order.save()

    for item in select_list:
        cart_item = Cart.objects.get(pk=item)
        cart_item.c_belong = True
        cart_item.c_order = order
        cart_item.save()

    data["order_num"] = order.id

    return JsonResponse(data)


def orderDetail(request):
    username = request.session.get("username")

    user = User.objects.get(u_name=username)

    orderid = request.GET.get("orderid")

    order = Order.objects.get(pk=orderid)
    # order = Order()
    goodsinfos = order.cart_set.all()

    data = {
        "user": user,
        "goodsinfos": goodsinfos,
        "orderid":orderid,
    }

    return render(request, 'app/market/order/order_detail.html', context=data)


def pay(request):
    orderid = request.GET.get("orderid")
    order = Order.objects.get(pk=orderid)
    order.o_status = 1
    order.save()
    data = {
        "status":"200",
        "msg":"ok",
    }
    return JsonResponse(data)


def notPayList(request):
    username = request.session.get("username")

    data = getOrders(username,0)

    return render(request,'app/market/order/order_list.html',context=data)


def notReceiveList(request):

    username = request.session.get("username")

    data = getOrders(username,1)

    return render(request,'app/market/order/order_list_not_receive.html',context=data)


def getOrders(username,status):
    if not username:
        return redirect(reverse("app:login"))
    user = User.objects.get(u_name=username)

    orderList = Order.objects.filter(o_user=user).filter(o_status=status)

    data = {
        "orderList": orderList,
    }
    return data
jkhjkjklkjkl
