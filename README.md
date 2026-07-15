# footwordstore

WhatsApp AI agent for Football World Store — built with N8N, Gemini and WAHA.

Estrutura de agentes multi-especialista para atendimento via WhatsApp no domínio de varejo de
artigos esportivos de futebol (camisas, chuteiras, acessórios). Ver
[docs/agents/INDEX.md](docs/agents/INDEX.md) para o índice de roteamento e
[docs/agents/n8n-setup.md](docs/agents/n8n-setup.md) para o guia de implementação no N8N.

O workflow importável está em [n8n/footwordstore-workflow.json](n8n/footwordstore-workflow.json) —
ver a seção "Importação rápida" do `n8n-setup.md` para colocar no ar.

O schema em [prisma/schema.prisma](prisma/schema.prisma) é a fonte de verdade do sistema de
estoque/vendas real da loja — os agentes devem usar os mesmos termos (categorias, status, canais)
definidos nele. Os endpoints reais que o n8n consome estão documentados em
[docs/agents/api-endpoints.md](docs/agents/api-endpoints.md).
