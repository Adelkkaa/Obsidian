> Intersection Observer используется для пересечения элемента с областью видимости

useInfiniteScroll:
```ts
import { MutableRefObject, useEffect, useRef } from 'react';

export interface UseInfiniteScrollOptions {
    callback?: () => void;
    triggerRef: MutableRefObject<HTMLElement>;
    wrapperRef: MutableRefObject<HTMLElement>;
}

export function useInfiniteScroll({ callback, wrapperRef, triggerRef }: UseInfiniteScrollOptions) {
    const observer = useRef<IntersectionObserver | null>(null);

    useEffect(() => {
        const wrapperElement = wrapperRef.current;
        const triggerElement = triggerRef.current;

        if (callback) {
            // Инициализируем опции для IntersectionObserver
            const options = {
                root: wrapperElement,
                rootMargin: '0px',
                threshold: 1.0,
            };

            // Инициализируем экземпляр класса IntersectionObserver, callback будет выполняться, когда current будет во viewport
            observer.current = new IntersectionObserver(([entry]) => {
                if (entry.isIntersecting) {
                    callback();
                }
            }, options);

            // Hазначаем current, за которым следим
            observer.current.observe(triggerElement);
        }

        return () => {
            if (observer.current && triggerElement) {
                // eslint-disable-next-line react-hooks/exhaustive-deps
                observer.current.unobserve(triggerElement);
            }
        };
    }, [callback, triggerRef, wrapperRef]);
}

```

- Хук использует IntersectionObserver для отслеживания пересечения элемента с областью видимости (viewport). Это позволяет определить, когда пользователь достиг нижней границы контейнера.
- В options определяются настройки для IntersectionObserver. root устанавливается на контейнер, в котором должен происходить скролл. Если скролл происходит в целом на странице, то в root устанавливается document.body. threshold: порог пересечения (1.0 означает полное пересечение), rootMargin: дополнительное пространство вокруг корневого элемента
- Создается новый экземпляр IntersectionObserver.Когда элемент `triggerElement` пересекается с viewport, вызывается `callback`.


Использование:

```tsx
import { classNames } from 'shared/lib/classNames/classNames';
import {
    memo, MutableRefObject, ReactNode, useRef,
} from 'react';
import { useInfiniteScroll } from 'shared/lib/hooks/useInfiniteScroll/useInfiniteScroll';
import cls from './Page.module.scss';

interface PageProps {
    className?: string;
    children: ReactNode;
    onScrollEnd?: () => void;
}

export const Page = memo((props: PageProps) => {
    const { className, children, onScrollEnd } = props;
    const wrapperRef = useRef() as MutableRefObject<HTMLDivElement>;
    const triggerRef = useRef() as MutableRefObject<HTMLDivElement>;

    useInfiniteScroll({
        triggerRef,
        wrapperRef,
        callback: onScrollEnd,
    });

    return (
        <section
            ref={wrapperRef}
            className={classNames(cls.Page, {}, [className])}
        >
            {children}
            <div ref={triggerRef} />
        </section>
    );
});

```

в onScrollEnd передаётся функция, которую необходимо выполнить, когда скролл достигнет конца страницы

