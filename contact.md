---
layout: page
title: "Contact"
permalink: /contact/
---



The best way to reach me is via <a id="maillink" data-mail="gershnik" data-domain="hotmail.com" href="#">e-mail</a>

If you prefer you can also message me on [LinkedIn](https://www.linkedin.com/in/gershnik/).

Very deliberately, I do not have a Facebook or a Twitter account.

<script>
let maillink = document.getElementById('maillink')  
maillink.innerHTML = maillink.getAttribute('data-mail')+'@'+ maillink.getAttribute('data-domain')
maillink.href = 'mailto:' + maillink.getAttribute('data-mail')+'@'+ maillink.getAttribute('data-domain')+'?subject=Hello there!'
</script>
