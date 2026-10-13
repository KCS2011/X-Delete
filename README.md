Script to delete all X / Twitter tweets. (Credits go to https://github.com/techleadhd, who let ChatGPT create the first version.). 

Updated to add a MAX_AGE_DAYS to keep some recents posts

### How to run:
- Open Chrome and go to your X profile (https://x.com/USERNAME/with_replies)
- Open Chrome Developer Console. Copy/Paste the code
- If needed adjust the MAX_AGE_DAYS setting
- Run!

```
const deleteAllTweets = async () => {
  const MAX_AGE_DAYS = 7; // only remove tweets older than this
  const cutoff = Date.now() - MAX_AGE_DAYS * 24 * 60 * 60 * 1000;

  const processed = new Set();
  const selectors = {
    tweet: '[data-testid="tweet"]',
    caret: '[data-testid="caret"]',
    menuItem: '[role="menuitem"]',
    deleteConfirm: '[data-testid="confirmationSheetConfirm"]',
    unretweet: '[data-testid="unretweet"]',
    unretweetConfirm: '[data-testid="unretweetConfirm"]'
  };

  let deleted = 0, skipped = 0;

  const delay = ms => {
    const jitter = ms * 0.25;
    const actual = ms + (Math.random() * jitter * 2 - jitter);
    return new Promise(resolve => setTimeout(resolve, actual));
  };

  // Returns the tweet's date, or null if it has none (e.g. promoted posts)
  const getTweetDate = tweet => {
    const time = tweet?.querySelector('time[datetime]'); // first = outer tweet, not a quoted one
    return time ? new Date(time.getAttribute('datetime')) : null;
  };

  const closeMenu = () =>
    document.body.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', bubbles: true }));

  const getButtons = () =>
    Array.from(document.querySelectorAll(`${selectors.tweet} ${selectors.caret}`))
      .filter(b => !processed.has(b));

  const scrollToEnd = () =>
    window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' });

  const attemptDelete = async button => {
    processed.add(button);
    const tweet = button.closest(selectors.tweet);
    const date = getTweetDate(tweet);

    // Age check: skip tweets without a date or newer than the cutoff
    if (!date || date.getTime() > cutoff) {
      skipped++;
      return false;
    }

    try {
      button.click();
      await delay(250);

      const menuItems = Array.from(document.querySelectorAll(selectors.menuItem));
      const deleteOption = menuItems.find(item =>
        /^(Delete|Verwijderen)$/i.test(item.textContent.trim())
      );

      if (deleteOption) {
        deleteOption.click();
        await delay(250);
        const confirm = document.querySelector(selectors.deleteConfirm);
        if (confirm) {
          confirm.click();
          deleted++;
          console.log(`Deleted tweet from ${date.toLocaleDateString()} (total: ${deleted})`);
          await delay(2000);
          return true;
        }
      } else {
        closeMenu(); // no Delete option (e.g. a retweet) – close the menu
        await delay(250);
      }

      const unretweet = tweet?.querySelector(selectors.unretweet);
      if (unretweet) {
        unretweet.click();
        await delay(250);
        const confirm = document.querySelector(selectors.unretweetConfirm);
        if (confirm) {
          confirm.click();
          deleted++;
          console.log(`Unretweeted post from ${date.toLocaleDateString()} (total: ${deleted})`);
          await delay(2000);
          return true;
        }
      }
    } catch (err) {
      console.error('Error attempting to delete/unretweet:', err);
    }
    return false;
  };

  console.log(`Removing tweets older than ${new Date(cutoff).toLocaleString()}`);

  while (true) {
    const buttons = getButtons();
    if (!buttons.length) {
      scrollToEnd();
      await delay(5000);
      if (!getButtons().length) break;
      continue;
    }

    for (const button of buttons) {
      const acted = await attemptDelete(button);
      if (acted) await delay(500); // cooldown only after an actual deletion
    }

    scrollToEnd();
    await delay(2000);
  }

  console.log(`Done. Removed: ${deleted}, skipped (newer than ${MAX_AGE_DAYS} days): ${skipped}`);
};

deleteAllTweets().catch(err => console.error('Script failed:', err));
```
<img width="813" alt="Screenshot 2023-12-31 at 7 18 45 PM" src="https://github.com/techleadhd/XDelete/assets/61847557/473165c5-9b7c-4065-98fd-5856fcbfb3a8">
