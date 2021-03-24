---
layout: page
title: "Contact"
permalink: /contact/
---



The best way to reach me is via <a id="maillink" data-mail="gershnik" data-domain="hotmail.com" href="#">e-mail</a>

If you prefer you can also message me on [LinkedIn](https://www.linkedin.com/in/gershnik/).

While, unfortunately, I do have a Facebook account, it is strictly reserved for immediate family only. Friend requests there will be ignored.

<script>
let maillink = document.getElementById('maillink')  
maillink.innerHTML = maillink.getAttribute('data-mail')+'@'+ maillink.getAttribute('data-domain')
maillink.href = 'mailto:' + maillink.getAttribute('data-mail')+'@'+ maillink.getAttribute('data-domain')+'?subject=Hello there!'
</script>
