# Website cart patch (corner-shop/index.html)

Five edits. Everything is inside the `<script>` at the bottom of the file.

## 1. Replace `saveCart`

Find:
```js
function saveCart() { try { localStorage.setItem("cart", JSON.stringify(cart)); } catch (e) {} updateCount(); }
```
Replace with:
```js
function saveCart() { try { if (user) localStorage.removeItem("cart"); else localStorage.setItem("cart", JSON.stringify(cart)); } catch (e) {} updateCount(); }
```

## 2. Add this block right after the `function cartTotal() {...}` line

```js
/* ---------- cart sync (Supabase) ---------- */
let cartChan = null, lastPush = 0;
async function pushCart(id) {
  if (!user) return;
  lastPush = Date.now();
  const qty = cart[id];
  if (!qty) { await sb.from("cart_items").delete().eq("user_id", user.id).eq("product_id", String(id)); return; }
  const p = products.find(x => String(x.id) === String(id));
  if (!p) return;
  await sb.from("cart_items").upsert({ user_id: user.id, product_id: String(id), name: p.name, price: p.price_ngn, quantity: qty }, { onConflict: "user_id,product_id" });
}
async function refreshCart() {
  const { data } = await sb.from("cart_items").select("*").eq("user_id", user.id);
  cart = {};
  (data || []).forEach(r => cart[r.product_id] = r.quantity);
  saveCart();
}
async function syncOnLogin() {
  if (!products.length) { const { data } = await sb.from("products").select("*").order("id"); products = data || []; }
  const guest = { ...cart };
  await refreshCart();
  for (const id of Object.keys(guest)) if (!(id in cart)) { cart[id] = guest[id]; await pushCart(id); }
  saveCart();
}
function subscribeCart() {
  unsubscribeCart();
  cartChan = sb.channel("cart-" + user.id)
    .on("postgres_changes", { event: "*", schema: "public", table: "cart_items", filter: "user_id=eq." + user.id }, async () => {
      if (Date.now() - lastPush < 1500) return;
      await refreshCart();
      if (location.hash === "#/checkout") renderCheckout();
    }).subscribe();
}
function unsubscribeCart() { if (cartChan) { sb.removeChannel(cartChan); cartChan = null; } }
```

## 3. Replace the four cart lines in the click handler

Find:
```js
  else if (act === "add") { cart[id] = (cart[id] || 0) + 1; saveCart(); toast("Added to cart"); }
  else if (act === "inc") { cart[id]++; saveCart(); renderCheckout(); }
  else if (act === "dec") { cart[id]--; if (cart[id] <= 0) delete cart[id]; saveCart(); renderCheckout(); }
  else if (act === "rm") { delete cart[id]; saveCart(); renderCheckout(); }
```
Replace with:
```js
  else if (act === "add") { cart[id] = (cart[id] || 0) + 1; saveCart(); pushCart(id); toast("Added to cart"); }
  else if (act === "inc") { cart[id]++; saveCart(); pushCart(id); renderCheckout(); }
  else if (act === "dec") { cart[id]--; if (cart[id] <= 0) delete cart[id]; saveCart(); pushCart(id); renderCheckout(); }
  else if (act === "rm") { delete cart[id]; saveCart(); pushCart(id); renderCheckout(); }
```

## 4. Clear the saved cart after an order

In `placeOrder`, find:
```js
  cart = {}; draft = { name: "", address: "" }; saveCart();
```
Replace with:
```js
  cart = {}; draft = { name: "", address: "" }; saveCart();
  lastPush = Date.now();
  await sb.from("cart_items").delete().eq("user_id", user.id);
```

## 5. Replace the `onAuthStateChange` block and add one line in `init`

Find the whole block:
```js
sb.auth.onAuthStateChange((_ev, session) => {
  const next = session ? session.user : null;
  if ((next && next.id) !== (user && user.id)) {
    user = next; renderAuth();
    if (products.length) route();
  }
});
```
Replace with:
```js
sb.auth.onAuthStateChange((_ev, session) => {
  const next = session ? session.user : null;
  if ((next && next.id) !== (user && user.id)) {
    user = next; renderAuth();
    setTimeout(async () => {
      if (user) { await syncOnLogin(); subscribeCart(); }
      else { cart = {}; saveCart(); unsubscribeCart(); }
      if (products.length) route();
    }, 0);
  }
});
```
Then, inside `init()`, find this line:
```js
  let back = null;
```
Add this line directly **above** it:
```js
  if (user && !cartChan) { await syncOnLogin(); subscribeCart(); }
```
