FROM redis:7-alpine

COPY redis.conf /usr/local/etc/redis/redis.conf

EXPOSE 6379

# Run redis with persistence enabled
CMD ["redis-server", "/usr/local/etc/redis/redis.conf"]
