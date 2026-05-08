---
tags: [ressource, mcp, integration]
---

# MCP — Model Context Protocol

Protokoll für Integration von externen Tools, Datenquellen und Services mit Claude.

## Was ist MCP?

MCP ist ein Standard-Protokoll, das es ermöglicht, Claude mit externen Systemen zu verbinden. Dadurch kann Claude auf Dateisysteme, Datenbanken, APIs und spezialisierten Tools zugreifen.

## MCP-Komponenten

- **Client** — Claude (oder andere LLMs)
- **Server** — Das externe System/Tool
- **Resources** — Daten, auf die zugegriffen wird
- **Tools** — Funktionen, die aufgerufen werden

## Einsatzbeispiele in meiner Arbeit

- **Dateisystem-Integration** — Zugriff auf Projektfiles
- **GitHub-Integration** — Repository-Zugriff
- **Datenbank-Verbindung** — Query und Manipulation
- **Custom-Tools** — Spezialisierten Services

## MCP Server für mein Setup

- **file-system** — Lokale Dateien
- **github** — GitHub Repositories
- **Custom Python Server** — Meine Spezial-Tools

## Integration mit Claude

1. MCP-Server konfigurieren
2. Mit Claude-Sesssion verbinden
3. Claude kann jetzt auf diese Ressourcen zugreifen
4. Workflows automatisieren

## Ressourcen

- [MCP Dokumentation](https://modelcontextprotocol.io)
- Meine MCP-Konfiguration (noch zu erstellen)
