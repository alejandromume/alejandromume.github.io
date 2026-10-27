---
layout: default
title: Nosotros
description: "Lava Digital: consultora de implementación de software e inteligencia artificial en Lanzarote. Desarrollo a medida, IA y ciberseguridad."
permalink: /nosotros/
---
<section class="hero">
  <p class="eyebrow">// nosotros</p>
  <h1>Software especializado, no plantillas genéricas</h1>
  <p class="lead">{{ site.author.name }} es {{ site.author.role | downcase }}, con sede en Lanzarote desde 2024. {{ site.author.bio }} Operamos bajo un principio simple: el software debe encajar en tu negocio, no al revés.</p>
</section>

<h2 class="section">qué hacemos</h2>
<div class="cards">
  <article class="card">
    <h2>Implementación de software</h2>
    <p>Aplicaciones empresariales, plataformas SaaS y sistemas de gestión a medida, adaptados a tu operativa real.</p>
  </article>
  <article class="card">
    <h2>Inteligencia artificial</h2>
    <p>Modelos LLM, sistemas RAG conectados a tus propios datos y automatización inteligente de tareas repetitivas.</p>
  </article>
  <article class="card">
    <h2>Ciberseguridad e infraestructura</h2>
    <p>Hardening de servidores, auditoría de vulnerabilidades y seguridad integrada desde el inicio del desarrollo, no añadida al final.</p>
  </article>
  <article class="card">
    <h2>Consultoría de procesos</h2>
    <p>Diagnóstico de tu operativa actual, identificación de cuellos de botella y un mapa de automatización, antes de escribir una sola línea de código.</p>
  </article>
</div>

<h2 class="section">por qué este blog</h2>
<p>Este espacio nace de nuestro trabajo de auditoría: cuando encontramos un fallo de seguridad real, lo reportamos primero al fabricante o a <a href="https://www.incibe.es/incibe-cert">INCIBE-CERT</a> para coordinar la divulgación, y después lo documentamos aquí con el nivel de detalle que el estado de cada caso permite. Puedes ver <a href="{{ '/todo/' | relative_url }}">todos los casos publicados</a> o explorar por <a href="{{ '/temas/' | relative_url }}">tema</a>. El resto de entradas son análisis de vulnerabilidades y noticias de actualidad que nos parecen relevantes, sin informe de por medio.</p>
<p>Los casos de auditoría los firma <strong>Alejandro Müller Medina</strong>, fundador de Lava Digital, que es quien encuentra y reporta cada fallo.</p>

<h2 class="section">hablemos</h2>
<p>Trabajamos con empresas de toda España, en remoto. Cuéntanos tu sector y tu reto operativo: evaluamos si podemos implementar una solución de software, IA o seguridad que lo resuelva, y te lo decimos claro, sin compromiso hasta que la propuesta encaje.</p>
<p>
  <a href="https://lavadigital.es">lavadigital.es</a>{% if site.author.email != "" %} · <a href="mailto:{{ site.author.email }}">{{ site.author.email }}</a>{% endif %}
</p>
