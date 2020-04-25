# EndpointSecurity API Documentation

This vault contains documentation and diagrams about Apple's EndpointSecurity API framework, which provides system event monitoring and security capabilities on macOS.

## Diagrams

### EndpointSecurity Event Types
![[assets/mermid/es-event-types.mermaid]]

This diagram illustrates the two main types of EndpointSecurity events: NOTIFY and AUTH events, and their key differences.

### EndpointSecurity Early Boot Process
![[assets/mermid/es-early-boot.mermaid]]

This sequence diagram shows how EndpointSecurity extensions are loaded during the early boot process and how they interact with the system.

### EndpointSecurity Client Flow
![[assets/mermid/es-client-flow.mermaid]]

This diagram demonstrates the typical flow of an EndpointSecurity client application, from initialization to event handling.

### EndpointSecurity Caching System
![[assets/mermid/es-caching.mermaid]]

This flowchart explains how the EndpointSecurity caching system works to improve performance and reduce redundant event processing.

### EndpointSecurity Architecture
![[assets/mermid/es-architecture.mermaid]]

This diagram provides an overview of the EndpointSecurity subsystem architecture and how events flow through the system.

## References

- [[Welcome|Getting Started]]

## Notes

- The diagrams are created using Mermaid syntax and can be viewed directly in Obsidian.
- You can click on any diagram to expand it for better viewing.
