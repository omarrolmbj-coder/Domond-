<!DOCTYPE html>
<html lang="ar">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>متجر تصميمات التيشرتات</title>
  <script src="https://cdn.jsdelivr.net/npm/react@18.2.0/umd/react.development.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/react-dom@18.2.0/umd/react-dom.development.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/babel-standalone@7.22.10/babel.min.js"></script>
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://js.stripe.com/v3/"></script>
  <style>
    body { font-family: 'Noto Sans Arabic', sans-serif; }
  </style>
</head>
<body dir="rtl" class="bg-gray-100">
  <div id="root"></div>
  <script type="text/babel">
    const { useState, useEffect } = React;

    // بيانات المنتجات
    const products = [
      { id: 1, name: "تيشرت أسود كلاسيك", price: 50, image: "https://via.placeholder.com/300x300?text=تيشرت+أسود" },
      { id: 2, name: "تيشرت أبيض مطبوع", price: 60, image: "https://via.placeholder.com/300x300?text=تيشرت+أبيض" },
      { id: 3, name: "تيشرت ملون", price: 55, image: "https://via.placeholder.com/300x300?text=تيشرت+ملون" },
    ];

    // مفتاح Stripe المنشور (استبدل بمفتاحك الحقيقي)
    const STRIPE_PUBLISHABLE_KEY = 'pk_test_your_publishable_key_here';

    // مفتاح PayPal Client ID (استبدل بمعرفك الحقيقي)
    const PAYPAL_CLIENT_ID = 'YOUR_PAYPAL_CLIENT_ID_HERE';

    // مكون المنتج
    function Product({ product, addToCart }) {
      return (
        <div className="bg-white rounded-lg shadow-md p-4 flex flex-col items-center">
          <img src={product.image} alt={product.name} className="w-48 h-48 object-cover mb-4" />
          <h3 className="text-lg font-semibold">{product.name}</h3>
          <p className="text-gray-600">السعر: {product.price} ريال</p>
          <button
            onClick={() => addToCart(product)}
            className="mt-4 bg-blue-500 text-white py-2 px-4 rounded hover:bg-blue-600"
          >
            أضف إلى السلة
          </button>
        </div>
      );
    }

    // مكون زر PayPal
    function PayPalButton({ total }) {
      useEffect(() => {
        const loadPayPalScript = () => {
          const script = document.createElement('script');
          script.src = `https://www.paypal.com/sdk/js?client-id=${PAYPAL_CLIENT_ID}&currency=SAR`;
          script.async = true;
          script.onload = () => {
            renderPayPalButton();
          };
          document.body.appendChild(script);
        };

        const renderPayPalButton = () => {
          if (window.paypal) {
            window.paypal.Buttons({
              createOrder: (data, actions) => {
                return actions.order.create({
                  purchase_units: [{
                    amount: {
                      value: total.toFixed(2) // تحويل إلى string مع .00
                    }
                  }]
                });
              },
              onApprove: (data, actions) => {
                return actions.order.capture().then((details) => {
                  alert(`تم إكمال الدفع بواسطة ${details.payer.name.given_name}`);
                  // هنا يمكن إضافة منطق النجاح، مثل مسح السلة
                });
              },
              onCancel: (data) => {
                alert('تم إلغاء الدفع');
              }
            }).render('#paypal-button-container');
          }
        };

        if (PAYPAL_CLIENT_ID !== 'YOUR_PAYPAL_CLIENT_ID_HERE') {
          loadPayPalScript();
        }
      }, [total]);

      if (PAYPAL_CLIENT_ID === 'YOUR_PAYPAL_CLIENT_ID_HERE') {
        return <p className="text-red-500">يرجى إضافة Client ID الخاص بـ PayPal لتفعيل الدفع.</p>;
      }

      return <div id="paypal-button-container"></div>;
    }

    // مكون سلة التسوق مع خيارات الدفع
    function Cart({ cartItems, removeFromCart }) {
      const total = cartItems.reduce((sum, item) => sum + item.price, 0);

      const handleStripeSubmit = async (event) => {
        event.preventDefault();
        // إنشاء جلسة Checkout عبر الخادم (يجب تنفيذ هذا على الخادم)
        const response = await fetch('/create-checkout-session', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            items: cartItems.map(item => ({ id: item.id, quantity: 1, price: item.price * 100 })), // السعر بالسنتات
          }),
        });

        const session = await response.json();

        // إعادة التوجيه إلى Stripe Checkout
        const stripe = window.Stripe(STRIPE_PUBLISHABLE_KEY);
        const { error } = await stripe.redirectToCheckout({
          sessionId: session.id
        });

        if (error) {
          console.error('خطأ في الدفع:', error);
          alert('حدث خطأ في معالجة الدفع: ' + error.message);
        }
      };

      return (
        <div className="bg-white rounded-lg shadow-md p-4 mb-8">
          <h2 className="text-xl font-bold mb-4">سلة التسوق</h2>
          {cartItems.length === 0 ? (
            <p className="text-gray-600">السلة فارغة</p>
          ) : (
            <div>
              {cartItems.map((item, index) => (
                <div key={index} className="flex justify-between items-center mb-2">
                  <span>{item.name} - {item.price} ريال</span>
                  <button
                    onClick={() => removeFromCart(index)}
                    className="text-red-500 hover:text-red-700"
                  >
                    إزالة
                  </button>
                </div>
              ))}
              <p className="text-lg font-semibold mt-4">الإجمالي: {total} ريال</p>
              <div className="mt-4 space-y-2">
                <form onSubmit={handleStripeSubmit}>
                  <button
                    type="submit"
                    className="w-full bg-green-500 text-white py-2 px-4 rounded hover:bg-green-600"
                  >
                    إتمام الدفع عبر Stripe
                  </button>
                </form>
                <PayPalButton total={total} />
              </div>
            </div>
          )}
        </div>
      );
    }

    // المكون الرئيسي
    function App() {
      const [cartItems, setCartItems] = useState([]);

      const addToCart = (product) => {
        setCartItems([...cartItems, product]);
      };

      const removeFromCart = (index) => {
        setCartItems(cartItems.filter((_, i) => i !== index));
      };

      return (
        <div className="container mx-auto p-4">
          <h1 className="text-3xl font-bold text-center mb-8">متجر تصميمات التيشرتات</h1>
          <Cart cartItems={cartItems} removeFromCart={removeFromCart} />
          <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
            {products.map((product) => (
              <Product key={product.id} product={product} addToCart={addToCart} />
            ))}
          </div>
        </div>
      );
    }

    // رندر التطبيق
    ReactDOM.render(<App />, document.getElementById('root'));
  </script>
</body>
</html>
