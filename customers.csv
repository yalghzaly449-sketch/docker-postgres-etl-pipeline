import pandas as pd
from sqlalchemy import create_engine, text
import time
import sys

def run_etl():
    db_url = 'postgresql://postgres:password@db:5432/etl_db'
    engine = None
    
    for attempt in range(1, 6):
        try:
            print("⏳ محاولة الاتصال بقاعدة البيانات...")
            engine = create_engine(db_url)
            with engine.connect() as conn:
                conn.execute(text("SELECT 1"))
            print("🎯 تم الاتصال بنجاح بقاعدة بيانات PostgreSQL!")
            break
        except Exception as e:
            if attempt == 5:
                print("❌ فشل الاتصال بعد 5 محاولات.")
                sys.exit(1)
            time.sleep(3)

    try:
        print("\n📥 [1/4] جاري قراءة ملفات الـ CSV وفحصها...")
        customers = pd.read_csv('customers.csv')
        products = pd.read_csv('products.csv')
        orders = pd.read_csv('orders.csv')

        print("🧹 [2/4] بدء تنظيف البيانات وحساب الـ total_price...")
        for df_name, df in [('Customers', customers), ('Products', products), ('Orders', orders)]:
            if df.isnull().values.any():
                df.dropna(inplace=True)

        merged = orders.merge(products, on='product_id', how='inner').merge(customers, on='customer_id', how='inner')
        merged['total_price'] = (merged['quantity'].astype(int) * merged['price'].astype(float)).round(2)
        final_orders = merged[['order_id', 'customer_id', 'product_id', 'quantity', 'total_price', 'order_date']]

        print("💾 [3/4] ضخ البيانات وتطبيق قيود الـ SQL الفنية...")
        customers.to_sql('customers', engine, if_exists='replace', index=False)
        products.to_sql('products', engine, if_exists='replace', index=False)
        final_orders.to_sql('orders', engine, if_exists='replace', index=False)

        with engine.connect() as conn:
            conn.execute(text("ALTER TABLE customers ADD PRIMARY KEY (customer_id);"))
            conn.execute(text("ALTER TABLE products ADD PRIMARY KEY (product_id);"))
            conn.execute(text("ALTER TABLE orders ADD PRIMARY KEY (order_id);"))
            conn.execute(text("ALTER TABLE orders ADD CONSTRAINT fk_customer FOREIGN KEY (customer_id) REFERENCES customers(customer_id);"))
            conn.execute(text("ALTER TABLE orders ADD CONSTRAINT fk_product FOREIGN KEY (product_id) REFERENCES products(product_id);"))
            conn.commit()
            
        print("✅ اكتمال الـ Pipeline بنجاح تام!")

        print("\n📊 [4/4] مخرجات الاستعلامات التحليلية الأربعة المطلوبة في مستقل:")
        print("="*50)
        
        # Q1: إجمالي المبيعات الكلية - تعديل آمن بدون أخطاء فورمت
        q1 = pd.read_sql("SELECT SUM(total_price) as total FROM orders", engine)
        total_val = float(q1['total'].iloc[0]) if q1['total'].iloc[0] is not None else 0.0
        print(f"1. إجمالي المبيعات الكلية: {total_val:,.2f} $")

        # Q2: المنتج الأكثر مبيعاً
        q2 = pd.read_sql("""
            SELECT p.product_name, SUM(o.quantity) as qty 
            FROM orders o JOIN products p ON o.product_id = p.product_id 
            GROUP BY p.product_name ORDER BY qty DESC LIMIT 1
        """, engine)
        print(f"2. المنتج الأكثر مبيعاً: {q2['product_name'].iloc[0]} (الكمية: {int(q2['qty'].iloc[0])})")

        # Q3: مبيعات كل مدينة
        q3 = pd.read_sql("""
            SELECT c.city, SUM(o.total_price) as sales 
            FROM orders o JOIN customers c ON o.customer_id = c.customer_id 
            GROUP BY c.city ORDER BY sales DESC
        """, engine)
        print(f"3. عوائد المبيعات حسب المدن:\n{q3.to_string(index=False)}")

        # Q4: حجم طلبات وإجمالي إنفاق كل عميل
        q4 = pd.read_sql("""
            SELECT c.customer_name, COUNT(o.order_id) as total_orders, SUM(o.total_price) as total_spent
            FROM orders o JOIN customers c ON o.customer_id = c.customer_id
            GROUP BY c.customer_name ORDER BY total_spent DESC
        """, engine)
        print(f"4. تقرير العملاء المفصل:\n{q4.to_string(index=False)}")
        print("="*50)

    except Exception as e:
        print(f"❌ حدث خطأ غير متوقع: {str(e)}")
        sys.exit(1)

if __name__ == "__main__":
    run_etl()
