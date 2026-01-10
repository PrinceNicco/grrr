FROM public.ecr.aws/x8v8d7g8/mars-base:latest
WORKDIR /app

ENV NODE_ENV=development
ENV NPM_CONFIG_PRODUCTION=false

COPY . .

RUN NODE_ENV=development pnpm install --production=false

CMD ["/bin/bash"]
