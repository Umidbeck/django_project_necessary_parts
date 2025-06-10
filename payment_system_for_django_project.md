# 💳 Django loyihasiga to'lov tizimlarini qo'shish yo'riqnomasi

## 🎯 1. O'zbekistondagi asosiy to'lov tizimlari

### Eng mashhur to'lov tizimlari:
- **Click** - Eng keng tarqalgan
- **Payme** - Uzcard bilan bog'liq
- **Paynet** - Keng tarqalgan
- **Upay** - Yangi tizim
- **Apelsin** - Mobil to'lovlar

## 🏗️ 2. Loyihada to'lov tizimini tashkil qilish

### A) Payments app yaratish
```bash
python manage.py startapp payments
```

### B) Payment modellari yaratish

```python
# apps/payments/models.py
from django.db import models
from django.contrib.auth.models import User
from apps.orders.models import Order
import uuid

class PaymentMethod(models.Model):
    """To'lov usullari"""
    PAYMENT_TYPES = [
        ('click', 'Click'),
        ('payme', 'Payme'),
        ('paynet', 'Paynet'),
        ('upay', 'Upay'),
        ('cash', 'Naqd pul'),
        ('card', 'Plastik karta'),
    ]
    
    name = models.CharField("Nomi", max_length=50)
    code = models.CharField("Kod", max_length=20, choices=PAYMENT_TYPES, unique=True)
    is_active = models.BooleanField("Faol", default=True)
    icon = models.ImageField("Ikonka", upload_to='payment_icons/', blank=True)
    description = models.TextField("Tavsif", blank=True)
    
    class Meta:
        verbose_name = "To'lov usuli"
        verbose_name_plural = "To'lov usullari"
    
    def __str__(self):
        return self.name


class Payment(models.Model):
    """To'lovlar"""
    STATUS_CHOICES = [
        ('pending', 'Kutilmoqda'),
        ('processing', 'Jarayonda'),
        ('success', 'Muvaffaqiyatli'),
        ('failed', 'Muvaffaqiyatsiz'),
        ('cancelled', 'Bekor qilingan'),
        ('refunded', 'Qaytarilgan'),
    ]
    
    # Asosiy ma'lumotlar
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name='payments')
    payment_method = models.ForeignKey(PaymentMethod, on_delete=models.CASCADE)
    
    # To'lov ma'lumotlari
    amount = models.DecimalField("Summa", max_digits=12, decimal_places=2)
    currency = models.CharField("Valyuta", max_length=3, default='UZS')
    status = models.CharField("Holat", max_length=20, choices=STATUS_CHOICES, default='pending')
    
    # Unique identifikatorlar
    payment_id = models.UUIDField("To'lov ID", default=uuid.uuid4, unique=True)
    transaction_id = models.CharField("Tranzaksiya ID", max_length=100, blank=True)
    external_id = models.CharField("Tashqi ID", max_length=100, blank=True)
    
    # Qo'shimcha ma'lumotlar
    gateway_response = models.JSONField("Gateway javobi", blank=True, null=True)
    failure_reason = models.TextField("Xatolik sababi", blank=True)
    
    # Vaqt ma'lumotlari
    created_at = models.DateTimeField("Yaratilgan", auto_now_add=True)
    updated_at = models.DateTimeField("Yangilangan", auto_now=True)
    paid_at = models.DateTimeField("To'langan", blank=True, null=True)
    
    class Meta:
        verbose_name = "To'lov"
        verbose_name_plural = "To'lovlar"
        ordering = ['-created_at']
    
    def __str__(self):
        return f"To'lov #{self.payment_id} - {self.amount} {self.currency}"


class PaymentLog(models.Model):
    """To'lov loglari"""
    payment = models.ForeignKey(Payment, on_delete=models.CASCADE, related_name='logs')
    action = models.CharField("Harakat", max_length=50)
    request_data = models.JSONField("So'rov ma'lumotlari", blank=True, null=True)
    response_data = models.JSONField("Javob ma'lumotlari", blank=True, null=True)
    ip_address = models.GenericIPAddressField("IP manzil", blank=True, null=True)
    created_at = models.DateTimeField("Yaratilgan", auto_now_add=True)
    
    class Meta:
        verbose_name = "To'lov logi"
        verbose_name_plural = "To'lov loglari"
    
    def __str__(self):
        return f"{self.payment.payment_id} - {self.action}"
```

## 🔧 3. Click to'lov tizimini integratsiya qilish

### A) Click API settings
```python
# apps/payments/click_settings.py
CLICK_SETTINGS = {
    'MERCHANT_ID': 'your_merchant_id',
    'SERVICE_ID': 'your_service_id', 
    'SECRET_KEY': 'your_secret_key',
    'MERCHANT_USER_ID': 'your_user_id',
    'BASE_URL': 'https://api.click.uz/v2/merchant/',
    'RETURN_URL': 'https://yoursite.com/payments/click/return/',
    'CANCEL_URL': 'https://yoursite.com/payments/click/cancel/',
}
```

### B) Click service yaratish
```python
# apps/payments/services/click_service.py
import requests
import hashlib
from django.conf import settings
from apps.payments.models import Payment, PaymentLog

class ClickService:
    def __init__(self):
        self.merchant_id = settings.CLICK_MERCHANT_ID
        self.service_id = settings.CLICK_SERVICE_ID
        self.secret_key = settings.CLICK_SECRET_KEY
        self.base_url = "https://api.click.uz/v2/merchant/"
    
    def create_payment_url(self, payment):
        """To'lov URL yaratish"""
        params = {
            'merchant_id': self.merchant_id,
            'service_id': self.service_id,
            'transaction_param': str(payment.payment_id),
            'amount': str(payment.amount),
            'return_url': f"{settings.SITE_URL}/payments/click/return/",
            'cancel_url': f"{settings.SITE_URL}/payments/click/cancel/",
        }
        
        # Imzo yaratish
        sign_string = (
            f"{params['merchant_id']}{params['service_id']}"
            f"{params['transaction_param']}{params['amount']}"
            f"{self.secret_key}"
        )
        params['sign'] = hashlib.md5(sign_string.encode()).hexdigest()
        
        # URL yaratish
        url = f"{self.base_url}create_invoice"
        query_string = "&".join([f"{k}={v}" for k, v in params.items()])
        
        return f"{url}?{query_string}"
    
    def check_payment_status(self, payment_id):
        """To'lov holatini tekshirish"""
        url = f"{self.base_url}payment_status"
        data = {
            'merchant_id': self.merchant_id,
            'service_id': self.service_id,
            'transaction_param': payment_id,
        }
        
        try:
            response = requests.post(url, json=data)
            return response.json()
        except Exception as e:
            return {'error': str(e)}
    
    def process_callback(self, request_data):
        """Callback ni qayta ishlash"""
        try:
            payment_id = request_data.get('transaction_param')
            payment = Payment.objects.get(payment_id=payment_id)
            
            # Imzoni tekshirish
            if self.verify_signature(request_data):
                if request_data.get('status') == 'success':
                    payment.status = 'success'
                    payment.transaction_id = request_data.get('transaction_id')
                    payment.paid_at = timezone.now()
                else:
                    payment.status = 'failed'
                    payment.failure_reason = request_data.get('error_note', '')
                
                payment.gateway_response = request_data
                payment.save()
                
                # Log saqlash
                PaymentLog.objects.create(
                    payment=payment,
                    action='callback_received',
                    response_data=request_data
                )
                
                return True
            
        except Payment.DoesNotExist:
            return False
        except Exception as e:
            return False
    
    def verify_signature(self, data):
        """Imzoni tekshirish"""
        received_sign = data.get('sign')
        
        # Imzo yaratish
        sign_string = (
            f"{data.get('merchant_id')}{data.get('service_id')}"
            f"{data.get('transaction_param')}{data.get('amount')}"
            f"{self.secret_key}"
        )
        calculated_sign = hashlib.md5(sign_string.encode()).hexdigest()
        
        return received_sign == calculated_sign
```

## 📝 4. Views yaratish

```python
# apps/payments/views.py
from django.shortcuts import render, redirect, get_object_or_404
from django.http import JsonResponse, HttpResponse
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_http_methods
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from django.utils import timezone
import json

from apps.orders.models import Order
from apps.payments.models import Payment, PaymentMethod
from apps.payments.services.click_service import ClickService

@login_required
def create_payment(request, order_id):
    """To'lov yaratish"""
    order = get_object_or_404(Order, id=order_id, customer__user=request.user)
    
    if order.is_paid:
        messages.error(request, "Bu buyurtma allaqachon to'langan!")
        return redirect('order-detail', order_id=order.id)
    
    if request.method == 'POST':
        payment_method_id = request.POST.get('payment_method')
        payment_method = get_object_or_404(PaymentMethod, id=payment_method_id)
        
        # To'lov yaratish
        payment = Payment.objects.create(
            order=order,
            payment_method=payment_method,
            amount=order.final_amount,
            status='pending'
        )
        
        if payment_method.code == 'click':
            # Click to'lov URL yaratish
            click_service = ClickService()
            payment_url = click_service.create_payment_url(payment)
            return redirect(payment_url)
        
        elif payment_method.code == 'payme':
            # Payme integratsiyasi
            return redirect('payme-payment', payment_id=payment.payment_id)
        
        elif payment_method.code == 'cash':
            # Naqd to'lov
            messages.success(request, "Buyurtma qabul qilindi! Naqd to'lov kurier orqali.")
            return redirect('order-detail', order_id=order.id)
    
    # To'lov usullarini ko'rsatish
    payment_methods = PaymentMethod.objects.filter(is_active=True)
    
    context = {
        'order': order,
        'payment_methods': payment_methods,
    }
    
    return render(request, 'payments/create_payment.html', context)

@csrf_exempt
@require_http_methods(["POST"])
def click_callback(request):
    """Click callback"""
    try:
        data = json.loads(request.body)
        click_service = ClickService()
        
        if click_service.process_callback(data):
            return JsonResponse({'status': 'success'})
        else:
            return JsonResponse({'status': 'error'}, status=400)
    
    except Exception as e:
        return JsonResponse({'status': 'error', 'message': str(e)}, status=500)

def payment_success(request):
    """To'lov muvaffaqiyatli"""
    payment_id = request.GET.get('payment_id')
    
    if payment_id:
        try:
            payment = Payment.objects.get(payment_id=payment_id)
            return render(request, 'payments/success.html', {'payment': payment})
        except Payment.DoesNotExist:
            pass
    
    return render(request, 'payments/success.html')

def payment_cancel(request):
    """To'lov bekor qilindi"""
    return render(request, 'payments/cancel.html')

def payment_status(request, payment_id):
    """To'lov holatini tekshirish"""
    try:
        payment = get_object_or_404(Payment, payment_id=payment_id)
        
        # Agar pending bo'lsa, holatni yangilash
        if payment.status == 'pending':
            click_service = ClickService()
            status_data = click_service.check_payment_status(payment_id)
            
            if status_data.get('status') == 'success':
                payment.status = 'success'
                payment.paid_at = timezone.now()
                payment.save()
        
        return JsonResponse({
            'status': payment.status,
            'amount': str(payment.amount),
            'created_at': payment.created_at.isoformat(),
        })
    
    except Payment.DoesNotExist:
        return JsonResponse({'error': 'To\'lov topilmadi'}, status=404)
```

## 🌐 5. URLs sozlash

```python
# apps/payments/urls.py
from django.urls import path
from . import views

app_name = 'payments'

urlpatterns = [
    path('create/<int:order_id>/', views.create_payment, name='create-payment'),
    path('click/callback/', views.click_callback, name='click-callback'),
    path('success/', views.payment_success, name='success'),
    path('cancel/', views.payment_cancel, name='cancel'),
    path('status/<uuid:payment_id>/', views.payment_status, name='status'),
]
```

## 🎨 6. Templates yaratish

```html
<!-- templates/payments/create_payment.html -->
{% extends 'base.html' %}
{% load static %}

{% block title %}To'lov - Favvora{% endblock %}

{% block content %}
<div class="container">
    <div class="row justify-content-center">
        <div class="col-md-8">
            <div class="card">
                <div class="card-header">
                    <h4>Buyurtma uchun to'lov</h4>
                </div>
                <div class="card-body">
                    <!-- Buyurtma ma'lumotlari -->
                    <div class="order-summary mb-4">
                        <h5>Buyurtma #{{ order.order_number }}</h5>
                        <p><strong>Jami summa: {{ order.final_amount|floatformat:0 }} so'm</strong></p>
                    </div>
                    
                    <!-- To'lov usullari -->
                    <form method="post">
                        {% csrf_token %}
                        <div class="payment-methods">
                            <h6>To'lov usulini tanlang:</h6>
                            
                            {% for method in payment_methods %}
                            <div class="form-check mb-3">
                                <input class="form-check-input" type="radio" 
                                       name="payment_method" value="{{ method.id }}" 
                                       id="payment_{{ method.id }}">
                                <label class="form-check-label" for="payment_{{ method.id }}">
                                    {% if method.icon %}
                                        <img src="{{ method.icon.url }}" alt="{{ method.name }}" 
                                             width="30" height="30" class="me-2">
                                    {% endif %}
                                    {{ method.name }}
                                    {% if method.description %}
                                        <small class="text-muted d-block">{{ method.description }}</small>
                                    {% endif %}
                                </label>
                            </div>
                            {% endfor %}
                        </div>
                        
                        <button type="submit" class="btn btn-primary btn-lg w-100">
                            To'lashga o'tish
                        </button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

## ⚙️ 7. Settings.py ga qo'shish

```python
# settings.py
INSTALLED_APPS = [
    # ... boshqa ilovalar
    'apps.payments',
]

# Click sozlamalari
CLICK_MERCHANT_ID = config('CLICK_MERCHANT_ID')
CLICK_SERVICE_ID = config('CLICK_SERVICE_ID')
CLICK_SECRET_KEY = config('CLICK_SECRET_KEY')

# Payme sozlamalari  
PAYME_MERCHANT_ID = config('PAYME_MERCHANT_ID')
PAYME_SECRET_KEY = config('PAYME_SECRET_KEY')

# Site URL
SITE_URL = config('SITE_URL', default='http://localhost:8000')
```

## 🔐 8. Xavfsizlik choralari

```python
# apps/payments/middleware.py
class PaymentSecurityMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # IP adreslarni cheklash
        if request.path.startswith('/payments/'):
            allowed_ips = [
                '185.178.208.0/24',  # Click IP diapazoni
                '195.158.31.0/24',   # Payme IP diapazoni
            ]
            
            client_ip = self.get_client_ip(request)
            # IP tekshirish logikasi
        
        response = self.get_response(request)
        return response
    
    def get_client_ip(self, request):
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            ip = x_forwarded_for.split(',')[0]
        else:
            ip = request.META.get('REMOTE_ADDR')
        return ip
```

## 📊 9. Admin panel

```python
# apps/payments/admin.py
from django.contrib import admin
from .models import Payment, PaymentMethod, PaymentLog

@admin.register(PaymentMethod)
class PaymentMethodAdmin(admin.ModelAdmin):
    list_display = ['name', 'code', 'is_active']
    list_filter = ['is_active']
    search_fields = ['name', 'code']

@admin.register(Payment)
class PaymentAdmin(admin.ModelAdmin):
    list_display = ['payment_id', 'order', 'amount', 'status', 'payment_method', 'created_at']
    list_filter = ['status', 'payment_method', 'created_at']
    search_fields = ['payment_id', 'transaction_id', 'order__order_number']
    readonly_fields = ['payment_id', 'created_at', 'updated_at']
    
    fieldsets = (
        ('Asosiy ma\'lumotlar', {
            'fields': ('order', 'payment_method', 'amount', 'currency', 'status')
        }),
        ('Identifikatorlar', {
            'fields': ('payment_id', 'transaction_id', 'external_id')
        }),
        ('Qo\'shimcha', {
            'fields': ('gateway_response', 'failure_reason'),
            'classes': ['collapse']
        }),
        ('Vaqt', {
            'fields': ('created_at', 'updated_at', 'paid_at'),
            'classes': ['collapse']
        })
    )

@admin.register(PaymentLog)
class PaymentLogAdmin(admin.ModelAdmin):
    list_display = ['payment', 'action', 'created_at']
    list_filter = ['action', 'created_at']
    readonly_fields = ['created_at']
```

## 🚀 10. Qadamma-qadam jarayon

1. **Preparation** (Tayyorgarlik)
   - To'lov provayderi bilan shartnoma
   - Test environment sozlash
   - API kalitlari olish

2. **Development** (Ishlab chiqish)
   - Models yaratish
   - Services yozish
   - Views va URLs
   - Templates yaratish

3. **Testing** (Sinov)
   - Test to'lovlar
   - Callback'larni sinash
   - Xatoliklarni tekshirish

4. **Production** (Ishga tushirish)
   - Production kalitlari
   - SSL sertifikat
   - Monitoring

Bu yo'riqnoma orqali to'lov tizimini muvaffaqiyatli integratsiya qilasiz! 💪
