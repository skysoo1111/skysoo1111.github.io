---
layout: post
title: "LangChain Deep Agents 완전 가이드 - 계획하고 기억하고 위임하는 AI 에이전트의 등장"
date: 2026-03-20 15:00:00 +0900
categories: [AI]
tags: [langchain, deep-agents, langgraph, ai-agent, python, nvidia]
---

## 개요

LangChain이 Deep Agents를 공식 릴리스했다. 단순한 도구 호출 루프를 넘어, 스스로 계획을 세우고 컨텍스트를 관리하며 서브에이전트에게 작업을 위임하는 구조화된 에이전트 런타임이다. GitHub 공개 5시간 만에 9.9k 스타를 달성할 만큼 커뮤니티의 반응이 뜨거운데, 왜 이렇게 주목받는지 핵심 아키텍처와 실전 코드를 함께 살펴본다.

## 기존 에이전트의 한계와 Deep Agents의 해법

기존 LLM 에이전트의 근본적인 문제는 세 가지로 압축된다. 복잡한 작업을 체계적으로 분해하지 못하고, 긴 대화에서 컨텍스트가 넘쳐 중요한 정보를 잃어버리며, 하나의 에이전트가 모든 것을 처리하려다 품질이 떨어진다.

Deep Agents는 이 문제를 4가지 아키텍처 컴포넌트로 해결한다.

- **상세 시스템 프롬프트**: Claude Code의 프롬프트 설계 방식을 참고한 정교한 기본 프롬프트
- **계획 도구(write_todos)**: 복잡한 태스크를 단계별로 분해하는 todo list 기반 계획
- **서브에이전트 생성**: 독립된 컨텍스트에서 하위 작업을 수행하는 에이전트 위임
- **가상 파일시스템**: 대용량 컨텍스트를 파일로 저장하고 필요할 때만 읽어오는 메모리 관리

핵심은 **컨텍스트 격리**다. 서브에이전트가 별도의 컨텍스트 윈도우에서 실행되므로, 메인 에이전트의 컨텍스트가 오염되지 않는다. 연구, 코딩, 분석처럼 수십 단계가 필요한 장기 자율 작업에서 이 설계가 결정적인 차이를 만든다.

## 5분 만에 시작하는 Deep Agents 실전 코드

`pip install deepagents`로 바로 설치할 수 있다. LangGraph 위에 구축되어 있어 기존 LangChain 생태계와 자연스럽게 호환된다.

**Before: 기존 LangChain 에이전트**

```python
from langchain_anthropic import ChatAnthropic
from langgraph.prebuilt import create_react_agent

llm = ChatAnthropic(model="claude-sonnet-4-5-20250514")
agent = create_react_agent(llm, tools=[search_tool, write_tool])

# 단순 도구 호출 루프 - 복잡한 작업에서 컨텍스트 유실 발생
result = agent.invoke({"messages": [("user", "3개 기술 블로그의 최신 트렌드를 분석해서 보고서를 작성해")]})
```

**After: Deep Agents**

```python
from deepagents import create_deep_agent
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(model="claude-sonnet-4-5-20250514")

agent = create_deep_agent(
    llm,
    tools=[search_tool, write_tool],
    # 서브에이전트 생성, todo 계획, 파일시스템 접근이 자동 포함
)

# 에이전트가 스스로 계획을 세우고, 서브에이전트에게 각 블로그 분석을 위임
result = agent.invoke({"messages": [("user", "3개 기술 블로그의 최신 트렌드를 분석해서 보고서를 작성해")]})
```

`create_deep_agent()`가 반환하는 것은 컴파일된 LangGraph 그래프다. 즉, 도구 호출을 지원하는 모든 LLM과 호환되고, 기존 LangGraph 워크플로우에 그대로 통합할 수 있다.

서브에이전트에 커스텀 도구를 추가하고 싶다면 다음과 같이 설정한다.

```python
agent = create_deep_agent(
    llm,
    tools=[search_tool, write_tool],
    sub_agent_tools=[code_executor, db_query_tool],
    max_sub_agents=5,
    filesystem_enabled=True,
)

# 스트리밍으로 에이전트의 계획 수립 과정을 실시간 관찰
for event in agent.stream({"messages": [("user", "프로젝트 코드베이스를 분석하고 리팩토링 계획을 작성해")]}):
    print(event)
```

## NVIDIA 파트너십과 엔터프라이즈 확장

LangChain은 Deep Agents 릴리스와 함께 NVIDIA와의 전략적 파트너십도 발표했다. 핵심 결과물인 **AI-Q Blueprint**는 Deep Agents 위에 구축된 프로덕션급 딥 리서치 시스템으로, 딥 리서치 벤치마크 1위를 달성했다.

이 파트너십이 시사하는 점은 명확하다. Deep Agents가 단순한 오픈소스 실험이 아니라, 엔터프라이즈 프로덕션을 목표로 설계되었다는 것이다. LangGraph 런타임의 상태 관리, Deep Agents의 태스크 계획과 서브에이전트 기능, NVIDIA의 병렬 실행 인프라가 결합되면서, 프로토타입에서 프로덕션까지의 간극이 좁혀지고 있다.

성능 면에서도 주목할 만하다. Claude Sonnet 4.5 기반 Deep Agents가 Terminal Bench 2.0에서 **42.65%**를 기록했는데, 이는 동일 모델의 Claude Code와 동등한 수준이다. 오픈소스 프레임워크가 상용 제품과 대등한 성능을 보여준 셈이다.

## 정리

- **Deep Agents는 계획-메모리-위임의 세 축**으로 기존 도구 호출 에이전트의 근본적 한계를 해결한다.
- **`create_deep_agent()` 한 줄**로 시작할 수 있고, LangGraph와 완전 호환되어 기존 코드를 크게 바꿀 필요가 없다.
- **컨텍스트 격리**가 핵심 설계 원칙이다. 서브에이전트가 독립된 컨텍스트에서 동작하므로 장기 작업에서도 품질이 유지된다.
- **NVIDIA 파트너십**으로 엔터프라이즈 프로덕션 경로가 열렸고, 벤치마크에서 상용 도구급 성능을 입증했다.
- MIT 라이선스 오픈소스이므로 지금 바로 `pip install deepagents`로 시작해볼 수 있다.

## 참고 자료

- [Deep Agents - LangChain 공식 블로그](https://blog.langchain.com/deep-agents/)
- [LangChain Releases Deep Agents - MarkTechPost](https://www.marktechpost.com/2026/03/15/langchain-releases-deep-agents-a-structured-runtime-for-planning-memory-and-context-isolation-in-multi-step-ai-agents/)
- [langchain-ai/deepagents - GitHub](https://github.com/langchain-ai/deepagents)
- [LangChain x NVIDIA Enterprise Agentic AI Platform](https://blog.langchain.com/nvidia-enterprise/)
- [LangChain Deep Agents Release - Awesome Agents](https://awesomeagents.ai/news/langchain-deep-agents-release/)
