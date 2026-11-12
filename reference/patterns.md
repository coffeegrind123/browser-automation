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
start_browser()                 # always headed; headless is refused
navigate("https://site.com/results?page=1")

# Per page:
#   execute_javascript("const items = ...; return items")      ← extract data (blocks with return are fine)
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

Each step renders new content, so walk the tree again after each step (new
elements get new ids; ids of elements that left are `stale_id`).

```
# Step 1
get_interaction_tree()
fill_form(form_data={"#name": "John", "#email": "j@d.com"})
click(selector="<next_id>")
wait_for_element(selector="#address")      # the next step's first field

# Step 2
get_interaction_tree()
fill_form(form_data={"#address": "123 St", "#city": "NYC"})
click(selector="<next_id>")
wait_for_element(selector="#review")

# Step 3 — verify before final submit
get_interaction_tree()
screenshot()
click(selector="<submit_id>")
wait_for_network(timeout=15)
```

## Cookie-Based Session Persistence

Save cookies after login, restore in a new session to skip login.

```
# Session 1: log in (by hand if it has 2FA) and save the whole cookie jar
start_browser()
# ... login flow ...
export_cookies(file_path="~/.sessions/site.json")    # HttpOnly included, file mode 0600
stop_browser()

# Session 2: restore
start_browser()
import_cookies(file_path="~/.sessions/site.json")
navigate("https://site.com/dashboard")
stop_browser()
```

Or keep a whole profile: `start_browser(user_data_dir="/path/profile")`.

## Monitoring / Polling

Check a page periodically for changes.

```
start_browser()                 # always headed; headless is refused
navigate(url)
previous = get_text_content()

# Loop:
#   wait(60)                    ← poll interval
#   reload_page()
#   wait_for_network(timeout=10)
#   current = get_text_content()
#   if current != previous → screenshot(), alert, update previous

stop_browser()
```

## Table Data Extraction

Use JS to extract structured data from HTML tables.

```
execute_javascript("(function(){
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

The interaction tree covers the top document only. Same-origin iframes are
readable with JS; cross-origin ones are not (the browser forbids it).

```
# List iframes
execute_javascript("(function(){
    return Array.from(document.querySelectorAll('iframe')).map(function(f,i){
        return {index: i, src: f.src, id: f.id};
    });
})()")

# Read iframe content
execute_javascript("document.querySelectorAll('iframe')[0].contentDocument.body.textContent")
```

## Infinite Scroll

Scroll to bottom repeatedly until no new content loads.

```
# Loop:
#   height = execute_javascript("document.body.scrollHeight")
#   scroll(direction="down", amount=height)
#   wait_for_network(timeout=5)     ← the scroll triggers the lazy load
#   new_height = execute_javascript("document.body.scrollHeight")
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

### Do NOT add waits after navigation
```
# BAD — navigate() already waited for the document and the network
navigate("https://spa.com")
wait(3)
get_interaction_tree()

# GOOD — and if an SPA renders late, wait for something specific
navigate("https://spa.com")
wait_for_element(selector="main [role=list]")
get_interaction_tree()
```

### Do NOT reuse numeric IDs after a navigation
```
# BAD
tree1 = get_interaction_tree()  # id=5 is "Submit"
navigate(other_page)
click(selector="5")             # fails: stale_id (never a different element)

# GOOD
navigate(other_page)
tree2 = get_interaction_tree()  # new ids
click(selector="<new_id>")
```
