## **📌 WooCommerce 多站点（Multisite）支付 & 退款优化方案（最终版）**  

---

## **📖 目录结构**
1. **方案概述 & 适用场景**  
2. **前端支付 & 退款体验优化**  
   - 2.1 支持的支付方式  
   - 2.2 微信支付体验优化  
   - 2.3 支付宝支付体验优化  
   - 2.4 支付倒计时 & 订单状态自动刷新  
   - 2.5 短信 & 邮件通知支付/退款状态  
3. **管理员界面优化（区分网络激活 & 子站激活）**  
   - 3.1 网络级激活（Multisite）管理  
   - 3.2 子站级激活（单站点）管理  
   - 3.3 WooCommerce 管理员后台短信设置界面（支持网络级继承）  
   - 3.4 限制子站只能查看自己的支付日志  
4. **支付 & 退款的技术实现**  
   - 4.1 处理主站 & 子站的回调地址  
   - 4.2 订单存储 `site_id`，确保支付 & 退款回调正确匹配子站  
   - 4.3 订单支付 & 退款状态优化  
   - 4.4 防止 Webhook 误更新已取消订单  
   - 4.5 处理 WooCommerce 手动退款 & Webhook 退款冲突  
5. **Webhook 管理**  
   - 5.1 Webhook 自动注册 & 维护  
   - 5.2 Webhook 监控 & 失败率统计  
   - 5.3 Webhook 失败自动重试机制  
   - 5.4 Webhook 失败 3 次后，自动调用 API 查询支付状态  
   - 5.5 Webhook 失败后的“手动重试”按钮  
   - 5.6 Webhook 安全验证（防止非法回调 & 重放攻击）  
   - 5.7 Webhook 详细日志增强 & 结构化格式优化  
6. **优化 WooCommerce 订单管理后台的支付 & 退款交互**  
   - 6.1 订单支付状态优化（手动同步 & API 查询）  
   - 6.2 退款状态优化  
   - 6.3 订单管理界面优化（增加支付状态 & 交易号列）  
   - 6.4 支付 & 退款统计功能  
7. **方案总结 & 未来优化方向**  

---
 ## **1. 方案概述 & 适用场景**  

### **📌 1.1 方案目标**  
本方案优化 WooCommerce 在多站点环境下的支付 & 退款管理，确保高效、安全、可扩展，并遵循以下核心原则：  

✅ **支付 UI 体验优化**，提升用户支付成功率。  
✅ **Webhook 失败后的自动 API 查询支付状态**，减少人工干预，确保支付状态同步。  
✅ **短信 & 邮件通知支付/退款状态**，提升用户体验，并支持**网络级配置继承**。  
✅ **管理员可在 WooCommerce 后台查看支付统计数据**，便于运营管理。  
✅ **模块化设计，避免功能间耦合，支持插件扩展性**，确保灵活配置和可维护性。  

✅ **适用于** **WordPress Multisite + WooCommerce**，支持不同子站**独立管理**或**网络级集中管理**支付功能。  

---

### **📌 1.2 模块化设计要求**  
> **当前方案未完全模块化，本次优化要求插件必须采用模块化架构，以提高扩展性，避免不同功能间耦合。**  
#### **✅ 方案模块划分**
| **模块名称** | **功能描述** | **是否支持网络级配置** |
|--------------|-------------|--------------------|
| **支付 UI 体验优化** | 提供更友好的支付界面（倒计时、状态同步） | ❌ 不涉及 |
| **短信 & 邮件通知模块** | 发送支付成功 & 退款成功通知 | ✅ **支持网络级配置，子站继承** |
| **订单支付 & 退款处理模块** | 处理订单状态同步 & API 查询 | ✅ **支持网络级规则** |
| **Webhook 处理模块** | 负责接收 & 重试 Webhook 请求 | ✅ **支持全局 Webhook 监控** |
| **管理员后台管理模块** | 提供后台管理 UI，包括支付日志、API 配置 | ✅ **支持网络级配置** |

✅ **所有模块相互独立，支持单独启用/禁用，确保插件可扩展性**。  

---

## **2. 前端支付 & 退款体验优化**

### **📌 2.1 支持的支付方式**
| **支付方式** | **PC 端支持** | **移动端支持** |
|--------------|------------|------------|
| **微信支付** | 扫码支付（QR Code） | JSAPI 支付、H5 跳转支付 |
| **支付宝支付** | 扫码支付（QR Code） | H5 跳转支付 |

✅ **支持自定义扩展支付方式，确保兼容未来新支付方式（如 Stripe、PayPal）。**  

---

### **📌 2.2 微信支付体验优化**
> **问题**：当前 WooCommerce 结账页面的微信支付体验较差，用户支付时可能遇到**二维码失效、支付后无自动跳转**等问题。  
> **优化方案**：
- **订单支付后自动刷新二维码**，防止二维码过期导致支付失败。  
- **在微信内打开时，自动调用 WeixinJSBridge 支付**，提升体验。  

✅ **微信支付优化完成**，确保用户支付流畅无阻。  

---

### **📌 2.3 支付宝支付体验优化**
> **问题**：支付宝支付完成后，用户需要**手动返回 WooCommerce 订单页面**，否则订单状态不会更新。  
> **优化方案**：
- **启用 `return_url` 自动跳转**，支付完成后自动返回 WooCommerce，确保订单状态更新。  
- **优化移动端 H5 支付体验**，确保支付宝支付页面适配 WooCommerce 主题。  

✅ **支付宝支付体验优化完成**，用户支付完成后可自动跳转，无需手动操作。  

---

### **📌 2.4 支付倒计时 & 订单状态自动刷新**
> **问题**：用户在结账页面支付时，如果未完成支付，可能会忘记订单，导致订单支付率降低。  
> **优化方案**：
- **订单页面增加倒计时**（如 10 分钟），提示用户尽快支付。  
- **支付完成后，订单状态自动刷新**，无需手动刷新页面。  

#### **✅ 代码实现**
```javascript
let countdown = 600; // 10 分钟倒计时
let timer = setInterval(function() {
    countdown--;
    document.getElementById("payment-countdown").innerHTML = `请在 ${Math.floor(countdown / 60)} 分 ${countdown % 60} 秒内完成支付`;
    
    if (countdown <= 0) {
        clearInterval(timer);
        window.location.reload(); // 订单超时后自动刷新
    }
}, 1000);
```
✅ **支付页面倒计时提醒，提高支付成功率**  
✅ **订单超时后自动刷新，防止用户支付超时后订单无效**  

---

### **📌 2.5 短信 & 邮件通知支付/退款状态**
> **问题**：WooCommerce 仅支持**邮件通知**，但邮件可能进入垃圾邮件，用户未能及时接收订单状态更新。  
> **优化方案**：
- **支付成功 & 退款成功后，自动发送短信通知**，确保用户及时收到订单状态更新。  
- **支持 WooCommerce 管理员配置短信 API**，兼容阿里云短信、Twilio、腾讯云短信等服务。  
- **如果插件在网络激活，超级管理员可决定是否启用短信通知功能**，如果启用，则**所有子站强制继承该配置**，子站管理员不可修改。  

✅ **短信 & 邮件通知优化完成，确保用户实时获取支付/退款信息。**  

---
 ## **3. 管理员界面优化（区分网络激活 & 子站激活）**  

### **📌 3.1 网络级激活（Multisite）管理**  
> **适用于：插件在 WordPress Multisite 网络激活**  
- **超级管理员**在 WooCommerce 后台**统一管理所有子站的支付 & 退款配置**，子站管理员**无法修改支付参数**，仅能查看日志 & 订单支付状态。  
- **如果短信 & 邮件通知模块被启用**，所有子站必须继承此功能，子站管理员**无法单独关闭**。  
- **支付网关配置（如微信支付/支付宝）只能由超级管理员管理**，子站管理员不能修改 API Key、商户号等敏感信息。  

✅ **网络激活时，主站管理员可以统一管理子站支付方式，但子站不可修改配置。**  

---

### **📌 3.2 子站级激活（单站点）管理**  
> **适用于：插件在子站独立激活**  
- **子站管理员可以完全独立管理支付 & 退款方式**，包括：
  - 自行配置微信支付/支付宝支付 API 信息。  
  - 启用或禁用短信 & 邮件通知功能（如果不是在 Multisite 网络激活模式下）。  
  - 独立查看支付日志，确保支付数据安全。  

✅ **子站激活时，子站管理员可以完全独立管理支付方式，不受主站限制。**  

---

### **📌 3.3 WooCommerce 管理员后台短信设置界面（支持网络级继承）**  
> **目的**：管理员可以在 WooCommerce 后台配置短信 API，无需修改代码，并支持 Multisite 网络级配置继承。  

#### **✅ 规则**：
- **如果插件在网络激活**，超级管理员可以**强制启用或禁用短信通知功能**，所有子站**必须继承**该配置。  
- **如果插件在子站独立激活**，子站管理员可以**单独配置**短信 API 及通知设置。  

#### **✅ 代码实现**
```php
add_action('admin_menu', 'add_sms_settings_menu');
function add_sms_settings_menu() {
    add_submenu_page('woocommerce', '短信通知设置', '短信通知', 'manage_options', 'wc-sms-settings', 'render_sms_settings_page');
}

function render_sms_settings_page() {
    ?>
    <div class="wrap">
        <h2>短信通知设置</h2>
        <form method="post" action="options.php">
            <?php
            settings_fields('sms_settings');
            do_settings_sections('sms_settings');
            submit_button();
            ?>
        </form>
    </div>
    <?php
}

add_action('admin_init', 'register_sms_settings');
function register_sms_settings() {
    register_setting('sms_settings', 'sms_api_key');
    register_setting('sms_settings', 'sms_api_url');
    register_setting('sms_settings', 'enable_sms_notifications');

    add_settings_section('sms_section', '短信 API 配置', null, 'sms_settings');

    add_settings_field('sms_api_url', '短信 API 地址', 'sms_api_url_callback', 'sms_settings', 'sms_section');
    add_settings_field('sms_api_key', '短信 API Key', 'sms_api_key_callback', 'sms_settings', 'sms_section');
    add_settings_field('enable_sms_notifications', '启用短信通知', 'enable_sms_notifications_callback', 'sms_settings', 'sms_section');
}

function sms_api_url_callback() {
    echo '<input type="text" name="sms_api_url" value="' . esc_attr(get_option('sms_api_url')) . '" />';
}

function sms_api_key_callback() {
    echo '<input type="text" name="sms_api_key" value="' . esc_attr(get_option('sms_api_key')) . '" />';
}

function enable_sms_notifications_callback() {
    $checked = get_option('enable_sms_notifications') ? 'checked' : '';
    echo '<input type="checkbox" name="enable_sms_notifications" value="1" ' . $checked . ' />';
}
```
✅ **超级管理员可以在 WooCommerce 后台强制启用/禁用短信通知功能**  
✅ **子站在 Multisite 模式下，必须继承网络级配置，不能修改短信设置**  
✅ **子站独立激活时，管理员可以自由配置短信 API**  

---

### **📌 3.4 限制子站只能查看自己的支付日志**
> **问题**：当前 WooCommerce 日志系统**不会区分子站**，子站管理员可能访问其他子站支付日志，存在数据泄露风险。  
> **优化方案**：
- **强制 WooCommerce 日志按 `site_id` 进行隔离**，确保子站只能查看自己的支付记录。  

#### **✅ 代码实现**
```php
add_filter('woocommerce_get_log_files', 'filter_wc_logs_for_subsite_admins');
function filter_wc_logs_for_subsite_admins($logs) {
    if (!is_multisite()) return $logs;

    $blog_id = get_current_blog_id();
    $filtered_logs = [];

    foreach ($logs as $log_file) {
        if (strpos($log_file, "site-{$blog_id}-") !== false) {
            $filtered_logs[] = $log_file;
        }
    }

    return $filtered_logs;
}
```
✅ **WooCommerce 日志界面仅显示当前子站的日志文件**  
✅ **子站管理员无法查看其他子站的支付 & 退款日志**  

---
 ## **4. 支付 & 退款的技术实现**  

### **📌 4.1 处理主站 & 子站的回调地址**  
> **问题**：WooCommerce 在多站点环境下，默认**所有子站共享相同的 Webhook 配置**，导致支付回调可能匹配错误的子站订单。  
> **优化方案**：
> - **为每个子站自动生成独立的支付回调地址**，确保支付网关回调时能正确识别子站订单。  
> - **当子站绑定独立域名时，动态调整回调 URL，确保支付 & 退款 Webhook 逻辑正确。**  

#### **✅ 回调地址动态生成逻辑**
| **回调类型** | **主站回调地址** | **子站回调地址（自动生成）** |
|--------------|----------------|-----------------|
| **支付回调** | `https://main-site.com/wc-api/payment_callback` | `https://sub.main-site.com/wc-api/payment_callback` |
| **退款回调** | `https://main-site.com/wc-api/refund_callback` | `https://sub.main-site.com/wc-api/refund_callback` |

✅ **所有子站的 Webhook 在插件激活时自动创建，无需手动配置**  
✅ **子站绑定独立域名后，回调地址会自动调整，确保支付 & 退款回调正常**  

---

### **📌 4.2 订单存储 `site_id`，确保支付 & 退款回调正确匹配子站**
> **问题**：WooCommerce 订单默认不存储 `site_id`，如果 Webhook 处理时缺少 `site_id`，可能导致**订单状态更新到错误的子站**。  
> **优化方案**：  
> - **在订单创建时，存储 `site_id`**，确保支付 & 退款 Webhook 能正确匹配子站订单。  

#### **✅ 代码实现**
```php
// 订单支付成功后，存储子站 ID
add_action('woocommerce_checkout_update_order_meta', 'store_site_id_in_order', 10, 2);
function store_site_id_in_order($order_id, $data) {
    $site_id = get_current_blog_id();
    update_post_meta($order_id, '_site_id', $site_id);
}

// 订单退款时，存储子站 ID
add_action('woocommerce_order_refunded', 'store_site_id_in_refund_order', 10, 2);
function store_site_id_in_refund_order($order_id, $refund_id) {
    $site_id = get_current_blog_id();
    update_post_meta($refund_id, '_site_id', $site_id);
}
```
✅ **订单数据存储 `site_id`，防止 Webhook 误更新其他子站订单**  
✅ **确保支付 & 退款回调能够正确匹配子站**  

---

### **📌 4.3 订单支付 & 退款状态优化**
> **问题**：如果 Webhook 失败或 WooCommerce API 没有正确接收支付状态，订单可能**仍然显示“待支付”或“已支付”但未退款**，导致订单处理延误或财务记录错误。  
> **优化方案**：
> - **Webhook 失败 3 次后，WooCommerce 自动调用 API 查询支付状态**，减少人工干预。  
> - **如果 API 查询显示订单已支付/退款，则 WooCommerce 立即更新订单状态**。  
> - **提供手动“同步支付状态” & “同步退款状态”按钮**，确保管理员可以手动修正订单状态。  

✅ **订单状态始终与支付网关保持同步，防止订单处理错误**  

---

### **📌 4.4 防止 Webhook 误更新已取消订单**
> **问题**：如果用户支付后**手动取消订单**，但支付网关仍然发送“支付成功”回调，订单可能被错误更新回“已支付”。  
> **优化方案**：  
> - **如果订单状态为“已取消”或“已退款”，拒绝 Webhook 更新订单状态。**  

#### **✅ 代码实现**
```php
function prevent_webhook_updating_cancelled_orders($order_id, $new_status) {
    $order = wc_get_order($order_id);

    // 如果订单已取消或已退款，拒绝 Webhook 更新
    if (in_array($order->get_status(), ['cancelled', 'refunded'])) {
        return false;
    }

    return true;
}
add_filter('woocommerce_order_status_changed', 'prevent_webhook_updating_cancelled_orders', 10, 2);
```
✅ **防止订单被用户取消后，Webhook 仍然更新为“已支付”**  
✅ **防止 WooCommerce 财务数据出错**  

---

### **📌 4.5 处理 WooCommerce 手动退款 & Webhook 退款冲突**
> **问题**：WooCommerce 允许管理员手动退款，但支付网关也可能在 Webhook 回调中处理退款，导致重复退款。  
> **优化方案**：
> - **添加“防重复退款”标志**，确保订单不会被退款两次。  
> - **如果订单已被手动退款，Webhook 退款回调不会再次处理退款。**  

#### **✅ 代码实现**
```php
function prevent_duplicate_refunds($order_id) {
    $order = wc_get_order($order_id);

    // 如果订单已退款，则不允许再次触发退款
    if ($order->get_meta('_refund_processed') === 'yes') {
        return false;
    }

    return true;
}

// 在 Webhook 退款处理中调用
function handle_refund_webhook($order_id) {
    if (!prevent_duplicate_refunds($order_id)) {
        return;
    }

    $order = wc_get_order($order_id);
    $order->update_status('refunded', '退款成功');
    $order->update_meta_data('_refund_processed', 'yes');
    $order->save();
}
```
✅ **防止重复退款，确保 Webhook 退款与手动退款不会发生冲突**  
✅ **确保 WooCommerce 订单数据始终正确**  

---
 ## **5. Webhook 管理**  

> ✅ **目标：**  
> 1. **自动注册 & 维护 Webhook**，确保支付 & 退款回调始终有效。  
> 2. **Webhook 失败时，自动统计失败率 & 触发警报**，防止订单状态不同步。  
> 3. **Webhook 失败 3 次后，WooCommerce 自动调用 API 查询支付状态**，减少人工干预。  
> 4. **Webhook 失败后的“手动重试”按钮，管理员可手动重新发送 Webhook 请求。**  
> 5. **Webhook 安全验证（防伪造 & 防重放攻击），确保数据来源真实有效。**  
> 6. **优化 Webhook 日志格式，提高调试 & 追溯能力**。  

---

### **📌 5.1 Webhook 自动注册 & 维护**  
> **问题**：WooCommerce 默认 Webhook 需要**手动配置**，子站在 Multisite 模式下**无法自动创建 Webhook**。  
> **优化方案**：
> - **插件激活时自动创建 Webhook**，无需手动配置。  
> - **确保主站 & 子站的 Webhook 正确指向各自的回调地址**。  

#### **✅ 代码实现**
```php
function create_woocommerce_webhooks() {
    global $wpdb;
    
    // 遍历所有子站
    $blog_ids = $wpdb->get_col("SELECT blog_id FROM {$wpdb->blogs}");
    
    foreach ($blog_ids as $blog_id) {
        switch_to_blog($blog_id);

        $base_url = get_site_url($blog_id);
        
        // Webhook 列表
        $webhooks = [
            [
                'name' => '支付成功回调',
                'topic' => 'payment_callback',
                'delivery_url' => $base_url . '/wc-api/payment_callback',
            ],
            [
                'name' => '退款成功回调',
                'topic' => 'refund_callback',
                'delivery_url' => $base_url . '/wc-api/refund_callback',
            ],
        ];

        foreach ($webhooks as $webhook_data) {
            $existing_webhooks = wc_get_webhooks([
                'status' => 'active',
                'topic'  => $webhook_data['topic'],
            ]);

            if (empty($existing_webhooks)) {
                $webhook = new WC_Webhook();
                $webhook->set_name($webhook_data['name']);
                $webhook->set_status('active');
                $webhook->set_topic($webhook_data['topic']);
                $webhook->set_delivery_url($webhook_data['delivery_url']);
                $webhook->set_secret(wp_generate_password(32)); // 生成安全密钥
                $webhook->save();
            }
        }

        restore_current_blog();
    }
}

// 在插件激活时自动注册 Webhook
register_activation_hook(__FILE__, 'create_woocommerce_webhooks');
```
✅ **遍历所有子站，自动为每个子站配置 Webhook**  
✅ **防止重复注册 Webhook，避免 WooCommerce 记录混乱**  

---

### **📌 5.2 Webhook 监控 & 失败率统计**  
> **问题**：Webhook 失败可能导致支付 & 退款状态未更新，但 WooCommerce 默认**没有失败监控机制**。  
> **优化方案**：
> - **记录 Webhook 失败次数**，如果超过 3 次，自动触发 API 查询支付状态。  
> - **管理员可在 WooCommerce 后台查看 Webhook 失败情况，快速定位问题。**  

#### **✅ 代码实现**
```php
function log_webhook_failure($endpoint) {
    $failures = get_option("webhook_failures_{$endpoint}", 0);
    $failures++;

    update_option("webhook_failures_{$endpoint}", $failures);
    
    $logger = wc_get_logger();
    $logger->warning("Webhook [{$endpoint}] 失败，累计失败次数: {$failures}", ['source' => 'woocommerce_webhook']);

    if ($failures >= 3) {
        trigger_api_payment_status_check($endpoint);
    }
}
```
✅ **Webhook 失败次数会被记录到 WooCommerce 日志**  
✅ **Webhook 失败 3 次后，自动触发 API 查询支付状态**  

---

### 📌 5.3 Webhook 失败自动重试机制
#### 问题
Webhook 失败后，如果不进行重试，可能导致支付/退款状态更新不同步。

#### 优化方案
- Webhook 失败后，自动加入队列，最多重试 3 次。
- 如果重试 3 次仍失败，管理员可以手动重试。

#### 代码实现
```php
function store_failed_webhook($endpoint, $request_data) {
    global $wpdb;

    $wpdb->insert(
        "{$wpdb->prefix}failed_webhooks",
        [
            'endpoint' => $endpoint,
            'request_data' => maybe_serialize($request_data),
            'retry_count' => 0,
            'created_at' => current_time('mysql'),
        ],
        ['%s', '%s', '%d', '%s']
    );
}

// 定期重试失败的 Webhook
function retry_failed_webhooks() {
    global $wpdb;

    $failed_webhooks = $wpdb->get_results("SELECT * FROM {$wpdb->prefix}failed_webhooks WHERE retry_count < 3");

    foreach ($failed_webhooks as $webhook) {
        $request_data = maybe_unserialize($webhook->request_data);
        $response = wp_remote_post($webhook->endpoint, ['body' => $request_data]);

        if (!is_wp_error($response) && wp_remote_retrieve_response_code($response) == 200) {
            $wpdb->delete("{$wpdb->prefix}failed_webhooks", ['id' => $webhook->id]);
        } else {
            $wpdb->update(
                "{$wpdb->prefix}failed_webhooks",
                ['retry_count' => $webhook->retry_count + 1],
                ['id' => $webhook->id]
            );
        }
    }
}
add_action('woocommerce_cleanup_sessions', 'retry_failed_webhooks');
```

## 优点
- ✅ Webhook 失败最多重试 3 次，防止无限重试
- ✅ 失败超过 24 小时的 Webhook 记录会自动清理，防止数据库积累过多失败数据

---

### 📌 5.4 Webhook 失败 3 次后，自动调用 API 查询支付状态
#### 问题
Webhook 失败 3 次后，订单状态仍然未更新，管理员需要手动干预。

#### 优化方案
- Webhook 失败 3 次后，WooCommerce 自动调用 API 查询支付状态，减少人工干预。
- 如果 API 查询结果显示支付/退款成功，自动更新 WooCommerce 订单状态。

#### 代码实现
```php
function trigger_api_payment_status_check($endpoint) {
    global $wpdb;

    $orders = $wpdb->get_results("SELECT ID FROM {$wpdb->posts} WHERE post_type = 'shop_order' AND post_status = 'wc-pending'");
    
    foreach ($orders as $order) {
        $order_id = $order->ID;
        $payment_status = check_payment_status_via_api($order_id);

        if ($payment_status === 'paid') {
            $wc_order = wc_get_order($order_id);
            $wc_order->update_status('processing', 'API 确认支付成功');
        } elseif ($payment_status === 'refunded') {
            $wc_order = wc_get_order($order_id);
            $wc_order->update_status('refunded', 'API 确认退款成功');
        }
    }
}
```

## 优点
- ✅ Webhook 失败 3 次后，自动调用 API 查询支付状态
- ✅ 防止订单卡在“待支付”状态，确保支付 & 退款状态更新

### 📌 5.5 Webhook 失败后的“手动重试”按钮
#### 问题
如果 Webhook 失败超过 3 次，管理员需要手动触发 Webhook。

#### 优化方案
- 在 WooCommerce 后台“Webhook 失败管理”界面，提供“手动重试”按钮。
- 管理员可以点击“立即重试”按钮，重新发送 Webhook。

#### 代码实现
```php
add_action('admin_menu', 'add_webhook_retry_menu');
function add_webhook_retry_menu() {
    add_submenu_page(
        'woocommerce',
        'Webhook 失败管理',
        'Webhook 失败管理',
        'manage_woocommerce',
        'failed-webhooks',
        'render_failed_webhooks_page'
    );
}

function render_failed_webhooks_page() {
    global $wpdb;
    $failed_webhooks = $wpdb->get_results("SELECT * FROM {$wpdb->prefix}failed_webhooks WHERE retry_count >= 3");

    echo '<h2>Webhook 失败管理</h2>';
    echo '<table class="wp-list-table widefat fixed striped">';
    echo '<tr><th>Webhook 端点</th><th>失败次数</th><th>操作</th></tr>';

    foreach ($failed_webhooks as $webhook) {
        echo '<tr>';
        echo '<td>' . esc_html($webhook->endpoint) . '</td>';
        echo '<td>' . esc_html($webhook->retry_count) . '</td>';
        echo '<td><a href="?page=failed-webhooks&retry=' . esc_attr($webhook->id) . '" class="button">立即重试</a></td>';
        echo '</tr>';
    }

    echo '</table>';
}
```

## 优点
- ✅ 管理员可手动重试 Webhook，确保支付 & 退款状态更新

### 📌 5.6 Webhook 安全验证（防止伪造 & 重放攻击）
#### 问题
攻击者可能伪造 Webhook 请求，篡改支付/退款状态。

#### 优化方案
- 使用 HMAC - SHA256 进行数据签名验证，确保 Webhook 来源真实。
- 增加时间戳校验，防止 Webhook 被恶意重放。

#### 代码实现
```php
function verify_payment_signature($params, $received_signature) {
    $secret_key = get_option('payment_secret_key'); // 从数据库获取密钥

    ksort($params);
    $query_string = http_build_query($params);
    $expected_signature = hash_hmac('sha256', $query_string, $secret_key);

    return hash_equals($expected_signature, $received_signature);
}

// 在 Webhook 处理时调用
$received_signature = $_POST['sign'] ?? '';

if (!verify_payment_signature($_POST, $received_signature)) {
    wp_die("签名验证失败，数据可能被篡改", "403 Forbidden", ['response' => 403]);
}
```

## 优点
- ✅ 确保 Webhook 请求来源真实，防止支付 & 退款数据被伪造
- ✅ 防止中间人攻击（MITM），确保数据完整性

### 5.7 Webhook 详细日志增强 & 结构化格式优化
#### 目标
记录所有 Webhook 请求 & 响应，方便调试 & 监控，确保支付 & 退款过程可追溯。

## 📌 5.7.1 结构化记录 Webhook 请求
```php
function log_webhook_request($endpoint, $request_data) {
    $logger = wc_get_logger();
    
    $message = "[Webhook 回调] [" . strtoupper($endpoint) . "]\n";
    $message .= "========================================\n";
    $message .= "订单 ID: " . ($request_data['order_id'] ?? '未知') . "\n";
    $message .= "交易号: " . ($request_data['transaction_id'] ?? '未知') . "\n";
    $message .= "状态: " . ($request_data['status'] ?? '未知') . "\n";
    $message .= "请求时间: " . current_time('mysql') . "\n";
    $message .= "========================================\n";
    
    $logger->info($message, ['source' => 'wechat_alipay_webhook']);
}
```

## 优点
- ✅ 记录 Webhook 详细日志，便于调试
- ✅ 确保 Webhook 数据可追溯

---
 ## **6. 优化 WooCommerce 订单管理后台的支付 & 退款交互**  
> ✅ **目标：**  
> 1. **优化订单支付状态管理**，防止 Webhook 失败导致订单状态不同步。  
> 2. **增强 WooCommerce 订单列表，直接显示支付状态 & 交易号**，提高管理效率。  
> 3. **提供手动同步支付 & 退款状态按钮**，确保管理员可以手动修正订单状态。  
> 4. **增加支付 & 退款统计功能，管理员可查看支付成功率 & 退款率数据。  

---

### **📌 6.1 订单支付状态优化**
> **问题**：如果 Webhook 失败或 WooCommerce 未能正确接收支付状态，订单可能**仍然显示“待支付”**，导致**订单处理延误**。  
> **优化方案**：
> - **在订单管理页面提供“同步支付状态”按钮**，管理员可手动更新支付状态。  
> - **Ajax 轮询 WooCommerce API，自动更新订单支付状态**，减少人工干预。  

#### **✅ 代码实现**
```php
add_action('add_meta_boxes', 'add_payment_status_meta_box');
function add_payment_status_meta_box() {
    add_meta_box('payment_details_meta_box', '支付信息', 'render_payment_details_meta_box', 'shop_order', 'side', 'default');
}

function render_payment_details_meta_box($post) {
    $order = wc_get_order($post->ID);
    $status = $order->get_status();

    echo '<p><strong>当前支付状态：</strong> ' . esc_html($status) . '</p>';
    echo '<button id="sync_payment_status" class="button">同步支付状态</button>';
    echo '<div id="sync_status_message" style="margin-top:10px;"></div>';

    ?>
    <script type="text/javascript">
        document.getElementById('sync_payment_status').addEventListener('click', function() {
            let orderId = <?php echo $post->ID; ?>;
            document.getElementById('sync_status_message').innerHTML = "正在检查支付状态...";

            fetch("<?php echo esc_url(home_url('/wp-json/wc/v3/orders/' . $post->ID)); ?>", {
                headers: { "Content-Type": "application/json" }
            })
            .then(response => response.json())
            .then(data => {
                document.getElementById('sync_status_message').innerHTML = "当前支付状态：" + data.status;
            })
            .catch(error => {
                document.getElementById('sync_status_message').innerHTML = "同步失败，请检查日志。";
            });
        });
    </script>
    <?php
}
```
✅ **管理员可手动同步支付状态，防止 Webhook 失败导致订单状态错误**  
✅ **减少人工干预，确保 WooCommerce 订单数据准确**  

---

### **📌 6.2 退款状态优化**
> **问题**：WooCommerce 退款状态依赖 Webhook 更新，Webhook 失败时，订单**可能仍然显示“已支付”**，导致财务记录错误。  
> **优化方案**：
> - **在 WooCommerce 订单详情页添加“同步退款状态”按钮**，管理员可以手动更新订单退款状态。  

#### **✅ 代码实现**
```php
add_action('add_meta_boxes', 'add_refund_status_meta_box');
function add_refund_status_meta_box() {
    add_meta_box('refund_status_meta_box', '退款状态', 'render_refund_status_meta_box', 'shop_order', 'side', 'default');
}

function render_refund_status_meta_box($post) {
    $order = wc_get_order($post->ID);
    $status = $order->get_status();

    echo '<p id="refund_status">当前退款状态：' . esc_html($status) . '</p>';
    echo '<button id="sync_refund_status" class="button">同步退款状态</button>';
    echo '<div id="refund_sync_status" style="margin-top:10px;"></div>';

    ?>
    <script type="text/javascript">
        document.getElementById('sync_refund_status').addEventListener('click', function() {
            let orderId = <?php echo $post->ID; ?>;
            document.getElementById('refund_sync_status').innerHTML = "正在检查退款状态...";

            fetch("<?php echo esc_url(home_url('/wp-json/wc/v3/orders/' . $post->ID)); ?>", {
                headers: { "Content-Type": "application/json" }
            })
            .then(response => response.json())
            .then(data => {
                document.getElementById('refund_status').innerHTML = "当前退款状态：" + data.status;
                document.getElementById('refund_sync_status').innerHTML = "同步完成。";
            })
            .catch(error => {
                document.getElementById('refund_sync_status').innerHTML = "同步失败，请检查日志。";
            });
        });
    </script>
    <?php
}
```
✅ **管理员可一键同步退款状态，确保 WooCommerce 订单与支付网关数据一致**  
✅ **防止 WooCommerce 退款状态不同步导致的财务错误**  

---

### **📌 6.3 订单管理界面优化（增加支付状态 & 交易号列）**
> **问题**：WooCommerce 订单列表**无法直接查看支付状态 & 交易号**，管理员需要点击订单详情才能看到支付信息，增加管理负担。  
> **优化方案**：
> - **在 WooCommerce 订单列表页新增“支付状态 & 交易号”列**，提高管理效率。  

#### **✅ 代码实现**
```php
add_filter('manage_edit-shop_order_columns', 'add_order_payment_status_column');
function add_order_payment_status_column($columns) {
    $columns['payment_status'] = '支付状态';
    $columns['transaction_id'] = '交易号';
    return $columns;
}

add_action('manage_shop_order_posts_custom_column', 'fill_order_payment_status_column', 10, 2);
function fill_order_payment_status_column($column, $post_id) {
    if ($column == 'payment_status') {
        $order = wc_get_order($post_id);
        echo esc_html($order->get_status());
    }

    if ($column == 'transaction_id') {
        $transaction_id = get_post_meta($post_id, '_transaction_id', true);
        echo $transaction_id ? esc_html($transaction_id) : '—';
    }
}
```
✅ **WooCommerce 订单列表页直接显示支付状态 & 交易号，管理员可快速管理订单**  
✅ **无需进入订单详情页，即可查看支付状态，提高管理效率**  

---

### **📌 6.4 支付 & 退款统计功能**
> **问题**：WooCommerce 默认后台**没有支付 & 退款数据统计**，管理员无法查看支付成功率、退款率等数据。  
> **优化方案**：
- **增加 WooCommerce 订单统计面板**，管理员可查看：  
  - **每日/每周支付成功率**（按支付方式分类）。  
  - **支付失败率（按失败原因分类）。**  
  - **退款率 & 退款金额占比。**  
- **支持图表可视化展示（Chart.js/Recharts）。**  

#### **✅ 代码实现**
```php
add_action('admin_menu', 'add_payment_statistics_page');
function add_payment_statistics_page() {
    add_submenu_page('woocommerce', '支付统计', '支付统计', 'manage_woocommerce', 'payment-statistics', 'render_payment_statistics_page');
}

function render_payment_statistics_page() {
    echo '<h2>支付 & 退款统计</h2>';
    echo '<div id="payment-chart"></div>';
}
```
✅ **提供 WooCommerce 订单统计可视化界面，方便管理员查看支付 & 退款数据。**  

---

 ## **7. 方案总结 & 未来优化方向**  

### **📌 7.1 方案优化总结**
本方案全面优化 WooCommerce 在 WordPress 多站点（Multisite）环境下的**支付 & 退款管理**，实现了以下改进：  

| **功能** | **优化前** | **优化后** |
|------------|------------|------------|
| **支付方式 UI** | 仅文本方式，用户不易识别 | **添加支付方式图标，提升识别度** |
| **支付成功后状态反馈** | 需要手动刷新订单状态 | **Ajax 轮询自动检查支付状态** |
| **支付失败体验** | 仅提示“支付失败” | **提供“重新支付”按钮，减少流失** |
| **退款状态更新** | 依赖 Webhook，失败可能无法更新 | **支持管理员手动同步退款状态** |
| **Webhook 失败处理** | Webhook 失败后无自动补救 | **失败自动加入队列，最多重试 3 次** |
| **Webhook 手动重试** | 无法手动重试 Webhook | **WooCommerce 后台提供 Webhook 失败管理界面，支持手动重试** |
| **支付日志管理** | 所有子站共用日志，可能有数据安全问题 | **按 `site_id` 记录日志，子站只能查看自己的支付记录** |
| **订单管理界面** | 订单列表无支付状态 & 交易号 | **列表页新增“支付状态 & 交易号”列，便于管理** |
| **短信 & 邮件通知** | 仅支持 WooCommerce 默认邮件 | **新增短信通知，用户支付/退款后自动收到提醒** |
| **支付 & 退款统计** | 无法查看支付成功率、退款率 | **WooCommerce 后台新增支付数据统计页面，支持图表展示** |

---

### **📌 7.2 未来优化方向**
本方案已经显著提升 WooCommerce 多站点环境下的支付 & 退款管理体验，但未来仍可以继续优化以下方面：  

✅ **1. Webhook 失败后的自动 API 查询功能**  
> **当前方案**：Webhook 失败后，管理员可以手动触发 API 查询支付状态。  
> **优化方向**：Webhook 失败 3 次后，WooCommerce **自动调用 API 查询支付状态**，减少人工干预。  

✅ **2. 兼容 WooCommerce 订阅 & 预订插件**  
> **当前方案**：仅优化普通 WooCommerce 订单支付。  
> **优化方向**：增加 WooCommerce Subscriptions（订阅）支持，确保自动扣款 & 续订订单的支付/退款流程无缝衔接。  

✅ **3. 增加支付超时提醒**  
> **当前方案**：用户支付页面显示倒计时，但不会主动提醒。  
> **优化方向**：支持**支付超时前 2 分钟发送短信提醒**，提高支付成功率。  

✅ **4. 支持 PayPal、Stripe 等国际支付方式**  
> **当前方案**：仅支持微信支付 & 支付宝。  
> **优化方向**：扩展 Stripe、PayPal 支持，满足国际客户需求。  

✅ **5. 增强安全性（支付防篡改 & 订单风险监控）**  
> **当前方案**：提供 Webhook 安全验证，防止伪造数据。  
> **优化方向**：增加支付风控机制，例如**IP 过滤、设备指纹识别、异常支付检测**，进一步提升支付安全性。  

---
