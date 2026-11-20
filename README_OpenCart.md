# OpenCart REST API – Cart Module (Postman)

Postman collection for testing the **Cart module** of the OpenCart REST API against a **local OpenCart installation** (XAMPP).

## Requests Covered

| # | Method | Request | Purpose |
|---|--------|---------|---------|
| 1 | POST | Create Session/Token | Log in with the API user and get the `api_token` |
| 2 | POST | Add product to cart | Add a product with a quantity |
| 3 | GET  | Cart content | View all products currently in the cart |
| 4 | POST | Edit product quantity in cart | Update quantity of a cart item |
| 5 | POST | Remove product from cart | Remove an item from the cart |

> Run the requests **in this order**. Request 1 must run first, since every other request needs the token.

## Prerequisites

- [XAMPP](https://www.apachefriends.org/) with **Apache** and **MySQL** running
- OpenCart installed locally (e.g. `http://localhost/opencart/`)
- [Postman](https://www.postman.com/downloads/)
- [Node.js](https://nodejs.org/) and Newman (only for command-line and Jenkins runs)
- [Jenkins](https://www.jenkins.io/download/) (optional), to automate runs from CI
- At least one enabled product with stock in the store (note its `product_id`)

## OpenCart Setup (one-time)

1. Start **Apache** and **MySQL** from the XAMPP Control Panel.
2. Open the admin panel: `http://localhost/opencart/admin`
3. Go to **System → Users → API** and create (or edit) an API user.
   - Set **Status = Enabled**
   - Copy the **API Username** and **API Key**
4. Under the API user's **IP Addresses** tab, add the IP you are calling from (for local testing: `127.0.0.1` and/or `::1`).
5. Make sure the product you want to test is **enabled** and has stock.

## Postman Setup

1. Import the collection JSON from this folder (**Import → File**).
2. Create an environment (e.g. `OpenCart Local`) with these variables:

| Variable | Example value |
|----------|---------------|
| `base_url` | `http://localhost/opencart` |
| `api_username` | your API username |
| `api_key` | your API key |
| `api_token` | *(leave empty – set automatically by request 1)* |
| `product_id` | `28` |
| `quantity` | `2` |
| `cart_id` | *(item key from Cart content, used for edit/remove)* |

3. Select the environment in the top-right of Postman.

## Test Data

| Field | Value |
|-------|-------|
| Product ID | `28` |
| Model | Product 1 |
| Price | 100.00 |
| Stock quantity | 939 |
| Status | Enabled |

### How to find a product ID

- **Admin:** Catalog → Products → click Edit; the URL ends with `product_id=28`.
- **Storefront:** open the product page; the URL contains `product_id=28`.
- **Database:** phpMyAdmin → `oc_product` table → `product_id` column.

## How to Run

1. **Create Session/Token** – sends API username + key. Saves `api_token` to the environment (via a test script).
2. **Add product to cart** – sends `product_id` and `quantity`.
3. **Cart content** – confirm the product appears in the cart.
4. **Edit product quantity in cart** – change the quantity of the item.
5. **Remove product from cart** – remove the item, then run **Cart content** again to confirm it is gone.

You can also run the whole folder with **Collection Runner** (the order above is preserved).

### Sample token-saving script (Tests tab of request 1)

```javascript
const res = pm.response.json();
if (res.api_token) {
    pm.environment.set("api_token", res.api_token);
}
pm.test("Status code is 200", () => pm.response.to.have.status(200));
pm.test("Token is returned", () => pm.expect(res.api_token).to.be.a("string"));
```

## Sample Assertions Used

- Status code is `200`
- Response time is acceptable
- Response contains `success` message on add / edit / remove
- Cart content shows the expected product and quantity
- Cart is empty (or item missing) after remove

## Run from Command Line (Newman)

```bash
npm install -g newman
npm install -g newman-reporter-htmlextra
```

Export the collection and the `OpenCart Local` environment from Postman, then run:

```bash
newman run OpenCart_CartModule_collection.json -e OpenCart_Local.postman_environment.json -r cli,htmlextra
```

The HTML report is saved in the `newman` folder. XAMPP (Apache and MySQL) must be running before the run starts.

## Run with Jenkins (Freestyle Project)

Jenkins runs on the same machine as XAMPP, so `http://localhost/opencart` works from the build.

### Start Jenkins
1. Open cmd and run:
   ```
   cd C:\Jenkins
   java -jar jenkins.war
   ```
2. Wait for `Jenkins is fully up and running`.
3. Open `http://localhost:8080` and log in. On the first run, use the initial admin password from the cmd log or from `C:\Users\<username>\.jenkins\secrets\initialAdminPassword`, then install the suggested plugins and create your admin user.

### Create a Freestyle Project
1. Click **New Item**, enter a name (for example `OpenCart-Cart-Newman`), select **Freestyle project** and click **OK**.
2. Under **Source Code Management**, select **Git** and add this repository URL, with the branch `*/main`.
3. Under **Build Environment**, tick **Use secret text(s) or file(s)** and add two **Secret text** bindings so credentials stay out of the repo:
   - Variable `API_USERNAME` → Jenkins credential holding your OpenCart API username
   - Variable `API_KEY` → Jenkins credential holding your OpenCart API key
4. Under **Build Steps → Add build step → Execute Windows batch command**, add:
   ```
   call newman run OpenCart_CartModule_collection.json -e OpenCart_Local.postman_environment.json --env-var "api_username=%API_USERNAME%" --env-var "api_key=%API_KEY%" -r cli,htmlextra
   ```
   `call` is needed on Windows. Without it, the batch script stops after Newman runs.
5. Under **Post-build Actions → Archive the artifacts**, enter `newman/*.html` so the report is saved with each build.
6. Click **Save**.

### Run and View Results
1. Start **Apache** and **MySQL** in the XAMPP Control Panel.
2. In Jenkins, click **Build Now**.
3. Open the build under **Build History**:
   - **Console Output** shows the Newman test results.
   - **Build Artifacts** has the HTML report.
4. A build is green when all tests pass and red when any test fails.

### Stop Jenkins
Press `Ctrl+C` in the cmd window.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `Warning: You do not have permission to access the API!` | Add your IP (`127.0.0.1`, `::1`) in the API user's IP tab; confirm API user is enabled |
| Token not returned / login fails | Re-check API username and key |
| `Connection refused` | Apache is not running in XAMPP |
| `404 Not Found` | Check `base_url` and the OpenCart folder name in `htdocs` |
| Product not added | Product disabled, out of stock, or wrong `product_id` |
| Session/cart empty on next call | Token expired – run request 1 again |
| Jenkins build stops after the newman line | Use `call newman ...` in the batch command |
| Jenkins build fails with `Connection refused` | Start Apache and MySQL in XAMPP before building |

## Environment

- OpenCart: `<add your version, e.g. 3.0.3.x / 4.0.x>`
- Server: XAMPP (Apache + MySQL, local)
- Tool: Postman `<version>`

## Notes

- Endpoint routes differ slightly between OpenCart 3.x and 4.x, so use the routes already saved in this collection.
- Do **not** commit real credentials. Keep `api_username` and `api_key` in environment variables and export the environment without secret values.

## Author

**MD Yousuff** – QA Tester / SDET
GitHub: [MD-YOUSUFF](https://github.com/MD-YOUSUFF) · LinkedIn: [Mohammed Yousuff](https://www.linkedin.com/in/mohammed-yousuff-068484217)
