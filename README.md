# MerchSNSReport
Inventoried skus orders which settled from 11/1/2016 to last day of previous month (12168)
SELECT
  o.order_id,
  o.date_created                        AS Order_date,
  s.shipment_number,
  cc.cc_type,
  s.settled,
  (CASE WHEN SRC.ID = 16
    THEN translate(s.auth_code, 'INT', 'FAC')
   WHEN SRC.ID = 17
     THEN '[INL]'
   ELSE s.auth_code END)                   auth_code,
  SUM(CASE WHEN i.item_type_id IN (1, 2, NULL)
    THEN os.price * osh.quantity
      ELSE 0 END)                       AS SKUS,
  SUM(CASE WHEN i.item_type_id IN (1, 2, NULL)
    THEN os.price * osh.quantity
      ELSE 0 END) * (d.percent / 100.0) AS DISCOUNT,
  SUM(CASE WHEN i.item_type_id IN (1, 2, NULL)
    THEN osh.tax
      ELSE 0 END)                       AS SKUS_TAX,
  SUM(CASE WHEN i.item_type_id IN (1, 2, NULL) AND og.SHIPMENT_ID > 9822844
    THEN 4.95
    WHEN i.item_type_id IN (1, 2, NULL) AND og.SHIPMENT_ID >= 9575190
    THEN 5.95
    WHEN i.item_type_id IN (1, 2, NULL) AND og.SHIPMENT_ID > 7956466
    THEN 4.95
      WHEN i.item_type_id IN (1, 2, NULL) AND og.SHIPMENT_ID >= 7869506
        THEN 6.95
      WHEN i.item_type_id IN (1, 2, NULL) AND og.SHIPMENT_ID > 1091844
        THEN 4.95
      WHEN og.SHIPMENT_ID <= 1091844
        THEN 4.50
      ELSE 0 END)                       AS GIFTS,
  s.gift_tax                            AS gifts_tax,
  s.ship_cost                           AS SHIP,
  s.shipment_tax                        AS SHIP_TAX,
  s.gift_cert_amount                    AS REDEEMED,
  0                                        REISSUED,
  SUM(CASE WHEN i.item_type_id = 3
    THEN gc.amount * osh.quantity
      ELSE 0 END)                       AS GIFT_CERTIFICATE_AMOUNT,
  SUM(CASE WHEN i.item_type_id = 3
    THEN gc.amount * osh.quantity
      ELSE 0 END) * (d.percent / 100.0) AS GIFT_CERTIFICATE_DISCOUNT,
  ad.POSTAL_CODE                           SHIP_ZIP,
  SRC.NAME                                 source,
  (CASE WHEN (SRC.ID >= 0 AND SRC.ID <= 4) OR (SRC.ID >= 6 AND SRC.ID <= 15)
    THEN 'Customer'
   WHEN SRC.ID = 16
     THEN 'Facebook'
   WHEN SRC.ID = 5
     THEN 'Internal'
   WHEN SRC.ID = 17
     THEN 'International' END)             category,
  o.initials                               placeby,
  dc.DISCOUNT_COST
FROM (SELECT *
      FROM orders o
      WHERE
        1 = 1 AND o.is_cancelled = 0 AND o.source_id IN (0, 2, 3, 4, 16, 17) AND o.payment_id <> 4 AND o.is_fraud <> 2
        AND o.DATE_CREATED > TO_DATE('11/1/2016', 'MM/DD/YYYY')) o LEFT JOIN discount d ON o.discount_id = d.discount_id
  INNER JOIN credit_card cc ON o.cc_id = cc.cc_id
  INNER JOIN source src ON o.SOURCE_ID = SRC.ID
  INNER JOIN (SELECT *
              FROM shipment s
              WHERE 1 = 1 AND (s.is_drop_ship = 0 OR s.is_drop_ship IS NULL) AND s.is_cancelled = 0 AND
                    s.authorized IS NOT NULL AND s.SETTLED < date_trunc('month', current_date) AND
                    s.SETTLED > TO_DATE('11/1/2016', 'MM/DD/YYYY') AND
                    (s.SHIPPED_DATE IS NULL OR s.SHIPPED_DATE >= date_trunc('month', current_date))) s
    ON s.order_id = o.order_id
  INNER JOIN order_shipment osh ON s.SHIPMENT_ID = osh.SHIPMENT_ID
  INNER JOIN order_sku os ON osh.ORDER_SKU_ID = OS.ORDER_SKU_ID
  INNER JOIN sku ON os.sku = sku.sku
  INNER JOIN item i ON sku.item_id = i.item_id
  LEFT JOIN (SELECT SHIPMENT_ID
             FROM order_shipment_gift
             GROUP BY SHIPMENT_ID) og
    ON og.SHIPMENT_ID = s.SHIPMENT_ID
  LEFT JOIN gift_certificate gc ON os.order_sku_id = gc.order_sku_id AND osh.shipment_id = gc.shipment_id
  LEFT JOIN address ad ON ad.address_id = s.address_id
  INNER JOIN (SELECT
                sh.shipment_id,
                SUM(osh.quantity * (CASE WHEN vsk.DISCOUNT_FIXED > 0
                  THEN ABS(vsk.price_per_unit - vsk.DISCOUNT_FIXED)
                                    WHEN vsk.discount_percent > 0
                                      THEN ABS(vsk.price_per_unit * (((vsk.DISCOUNT_PERCENT / 100) - 1) * -1))
                                    ELSE ABS(vsk.price_per_unit) END)) DISCOUNT_COST
              FROM shipment sh, order_shipment osh, order_sku osk, vendor_sku vsk
              WHERE osh.SHIPMENT_ID = sh.SHIPMENT_ID AND osk.ORDER_SKU_ID = osh.ORDER_SKU_ID AND osk.SKU = vsk.SKU AND
                    sh.IS_CANCELLED = 0 AND sh.IS_HOLD = 0 AND sh.IS_TEMP_HOLD = 0 AND
                    sh.SETTLED < date_trunc('month', current_date) AND sh.SETTLED > TO_DATE('11/1/2016', 'MM/DD/YYYY')
              GROUP BY sh.SHIPMENT_ID) dc ON dc.shipment_id = s.shipment_id
GROUP BY o.order_id, o.date_created, s.shipment_number, cc.cc_type, s.settled, (CASE WHEN SRC.ID = 16
  THEN translate(s.auth_code, 'INT', 'FAC')
                                                                                WHEN SRC.ID = 17
                                                                                  THEN '[INL]'
                                                                                ELSE s.auth_code END), s.gift_tax,
  s.ship_cost, s.shipment_tax, s.gift_cert_amount, REISSUED, ad.POSTAL_CODE, SRC.NAME,
  (CASE WHEN (SRC.ID >= 0 AND SRC.ID <= 4) OR (SRC.ID >= 6 AND SRC.ID <= 15)
    THEN 'Customer'
   WHEN SRC.ID = 16
     THEN 'Facebook'
   WHEN SRC.ID = 5
     THEN 'Internal'
   WHEN SRC.ID = 17
     THEN 'International' END), o.initials, d.percent, dc.DISCOUNT_COST;
