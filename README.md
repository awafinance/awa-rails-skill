# Awa Rails para agentes

Instale a skill pública, conecte seu agente ao MCP e configure os fluxos de NFS-e, NF-e recebidas e notificações.

## Cole este prompt no seu agente

```text
Configure o Awa Rails para este agente seguindo o guia público:
https://raw.githubusercontent.com/awafinance/awa-rails-skill/main/skills/awa-rails/references/install.md

Instale a skill awa-rails de https://github.com/awafinance/awa-rails-skill, incluindo suas referências, preservando minha configuração atual.
Conecte o MCP em https://rails.awafinance.com/mcp. Me guie para criar a chave na aba Chaves de API e guardá-la como AWA_RAILS_API_KEY nas configurações de segredos; nunca neste chat.
Descubra as ferramentas, consulte as capacidades, confirme minha empresa e explique como emitir NFS-e, receber NFS-e e NF-e e fazer ciência e manifestação de NF-e.
Me ajude a configurar um receptor HTTPS de webhooks com assinatura verificada, deduplicação e notificações no serviço que eu escolher. Para recepção contínua de NFS-e, ative o worker somente com minha autorização e um NSU inicial explícito. Para NF-e, configure sincronização externa autorizada, respeitando os intervalos da SEFAZ.
Valide a conexão com consultas sem efeitos fiscais. Só emita, manifeste, publique um receptor ou envie notificações quando eu tiver autorizado essas ações. Diga o que está pronto e o que ainda depende de mim.
```

O agente identifica seu cliente e instala as instruções. Você faz login no Rails, escolhe a empresa e fornece credenciais pelo gerenciador de segredos.

- [Skill](skills/awa-rails/SKILL.md)
- [Instalação e configuração do MCP](skills/awa-rails/references/install.md)
- [Operações fiscais](skills/awa-rails/references/operations.md)
- [Webhooks e notificações](skills/awa-rails/references/webhooks.md)
- [Perfil tributário suportado](skills/awa-rails/references/tax-profile.md)

O MCP emite **NFS-e** e recebe **NFS-e** e **NF-e**. Não há emissão de NF-e pelo MCP. Consulte as capacidades ativas: operações reais de homologação e simulação são distintas. Emissão e eventos de NF-e em produção permanecem bloqueados; a recepção de NFS-e tem um piloto separado com allowlist explícita. Um webhook não inicia a consulta à SEFAZ nem acorda automaticamente um agente.

Este repositório contém somente a skill e suas referências públicas. A API e o app estão em https://rails.awafinance.com.
