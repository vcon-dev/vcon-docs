# 🧩 API

## Introduction

## Authorization

The Conserver API provides a token based authentication, controlled by the environment variable CONSERVER\_API\_TOKEN in the environment.   When not defined or empty, it is disabled.  To enable, define CONSERVER\_API\_TOKEN in the .env file

## vCon Management

{% swagger src="../.gitbook/assets/openapi.json" path="/vcon" method="get" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}

{% swagger src="../.gitbook/assets/openapi.json" path="/vcon" method="post" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}

{% swagger src="../.gitbook/assets/openapi.json" path="/vcon/{vcon_uuid}" method="get" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}

{% swagger src="../.gitbook/assets/openapi.json" path="/vcon/{vcon_uuid}" method="delete" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}



## Chain Management

Chains are series of links that process a vCon.  Before processing a vCon, be sure to load it.

{% swagger src="../.gitbook/assets/openapi.json" path="/vcon/ingress" method="post" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}

{% swagger src="../.gitbook/assets/openapi.json" path="/vcon/egress" method="get" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}

{% swagger src="../.gitbook/assets/openapi.json" path="/vcon/count" method="get" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}



## Configuration

{% swagger src="../.gitbook/assets/openapi.json" path="/config" method="get" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}

{% swagger src="../.gitbook/assets/openapi.json" path="/config" method="post" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}

{% swagger src="../.gitbook/assets/openapi.json" path="/config" method="delete" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}



## Dead Letter Queue

{% swagger src="../.gitbook/assets/openapi.json" path="/dlq/reprocess" method="post" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}

{% swagger src="../.gitbook/assets/openapi.json" path="/dlq" method="get" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}



## Lifecyle

{% swagger src="../.gitbook/assets/openapi.json" path="/index_vcons" method="get" %}
[openapi.json](../.gitbook/assets/openapi.json)
{% endswagger %}
