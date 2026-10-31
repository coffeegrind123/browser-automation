# Browser Automation Patterns

## Login Then Scrape

```
start_browser()
navigate("https://site.com/login")
get_interaction_tree()
type_text(text="user@email.com", selector="<username_id>")
type_text(text="password123", selector="<password_id>")
click(selector="<login_button_id>")
wait_for_network(timeout=10)
screenshot()                    # verify login succeeded
navigate("https://site.com/dashboard")
get_text_content()              # scrape authenticated content
stop_browser()
```

## Paginated Data Collection

```
start_browser(headless=true)
navigate("https://site.com/results?page=1")

# Per page:
#   execute_js("(function(){ ... return items; })()")  ← extract data
#   get_interaction_tree()                              ← find "Next" button
#   click(selector="<next_id>")                        ← paginate
#   call_tool("wait_for_network", {"timeout": 5})       ← the CLICK needs this
#   If no "Next" button in tree → done

stop_browser()
```

## SPA / Dynamic Content

SPAs load content asynchronously. The DOM mutates after initial load.

```
navigate("https://spa-app.com")
wait_for_element(selector=".content-loaded", timeout=15, visible=true)
get_interaction_tree()          # now the real content is available
```

## Network Request Interception

Monitor API calls triggered by user actions. Ideal for endpoint-based scraping.

```
clear_logs()                    # reset before action
click(selector="<trigger_id>")  # trigger the API call
wait_for_request(url_pattern="api/v2/data")
get_network_logs()              # see full request/response details
```

## Multi-Step Wizard / Checkout

Each step loads new content. **Must re-query interaction tree after each step.**

```
# Step 1
get_interaction_tree()
fill_form(json={"#name": "John", "#email": "j@d.com"})
click(selector="<next_id>")
wait_for_network()
wait(1)

# Step 2 — IDs have changed!
get_interaction_tree()
fill_form(json={"#address": "123 St", "#city": "NYC"})
click(selector="<next_id>")
wait_for_network()
wait(1)

# Step 3 — verify before final submit
get_interaction_tree()
screenshot()
click(selector="<submit_id>")
wait_for_network(timeout=15)
```

## Cookie-Based Session Persistence

Save cookies after login, restore in a new session to skip login.

```
# Session 1: Login and save
start_browser()
# ... login flow ...
cookies = get_cookies()         # save these
stop_browser()

# Session 2: Restore
start_browser()
set_cookies(cookies=saved_cookies)
navigate("https://site.com/dashboard")
stop_browser()
```

## Monitoring / Polling

Check a page periodically for changes.

```
start_browser(headless=true)
navigate(url)
previous = get_text_content()

# Loop:
#   wait(60)                    ← poll interval
#   reload_page()
#   wait(3)
#   current = get_text_content()
#   if current != previous → screenshot(), alert, update previous

stop_browser()
```

## Table Data Extraction

Use JS to extract structured data from HTML tables.

```
execute_js("(function(){
    var rows = document.querySelectorAll('table tbody tr');
    return Array.from(rows).map(function(row){
        var cells = row.querySelectorAll('td');
        return {
            col1: cells[0].textContent.trim(),
            col2: cells[1].textContent.trim(),
            col3: cells[2].textContent.trim()
        };
    });
})()")
```

## Modal / Popup Handling

Modals block underlying elements. Check for `r:"dlg"` region in interaction tree.

```
get_interaction_tree()
# Elements with r:"dlg" = modal content
# Find close button (usually X icon in dlg region)
click(selector="<close_button_id>")
wait(0.5)
# Or dismiss with Escape:
press_key(key="Escape")
wait(0.5)
get_interaction_tree()          # now main page is accessible
```

## Iframe Content

Iframes need JS to access their content.

```
# List iframes
execute_js("(function(){
    return Array.from(document.querySelectorAll('iframe')).map(function(f,i){
        return {index: i, src: f.src, id: f.id};
    });
})()")

# Read iframe content
execute_js("document.querySelectorAll('iframe')[0].contentDocument.body.textContent")
```

## Infinite Scroll

Scroll to bottom repeatedly until no new content loads.

```
# Loop:
#   height = execute_js("document.body.scrollHeight")
#   scroll(direction="down", amount=height)
#   wait_for_network(timeout=5)     ← the scroll triggers the lazy load
#   new_height = execute_js("document.body.scrollHeight")
#   if new_height == height → done (no new content)

get_text_content()              # extract all loaded content
```

## Anti-Patterns

### Do NOT use get_content() for discovery
```
# BAD — wastes tokens
get_content()

# GOOD — compact, structured, actionable
get_interaction_tree()
```

### Do NOT hardcode coordinates
```
# BAD — breaks on any layout change
mouse_click(x=350, y=720)

# GOOD — resilient
click(selector="<numeric_id>")
click(text="Submit Order")
```

### Do NOT skip waits after navigation
```
# BAD — element likely not in DOM yet
navigate("https://spa.com")
get_interaction_tree()          # partial or empty

# GOOD
navigate("https://spa.com")
get_interaction_tree()          # complete
```

### Do NOT reuse numeric IDs after page changes
```
# BAD — IDs regenerated on every call
tree1 = get_interaction_tree()  # id=5 is "Submit"
navigate(other_page)
click(selector="5")             # id=5 is now something else!

# GOOD
navigate(other_page)
tree2 = get_interaction_tree()  # fresh IDs
click(selector="<new_id>")
```
